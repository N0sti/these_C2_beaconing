# Pipeline ML et Dashboard SOC — Documentation complète

Ce document explique en détail le passage des captures réseau brutes (.pcap) jusqu'au dashboard de détection, avec le code, les choix de modèles, les paramètres et la lecture des résultats.

---

## Vue d'ensemble du pipeline

```
.pcap (paquets bruts)
   │
   ▼  extract_features_windowed.py
fenêtres de trafic + 12 statistiques (intervalle, taille...)
   │
   ▼  build_dataset_windowed.py
dataset.csv (1 ligne = 1 fenêtre de trafic, avec son label vérité-terrain)
   │
   ▼  train_model.py
model_final.joblib (règles apprises pour distinguer C2 / légitime)
   │
   ▼  init_db.py + app.py (au moment de la prédiction)
confidence_score → severity → ligne dans soc.db
   │
   ▼
dashboard (KPI rouge/orange/vert, requêtes SQL, graphes, flux live)
```

---

## 1. Extraction des features — `extract_features_windowed.py`

### Pourquoi des "flux" et pas des paquets

Un `.pcap` contient des paquets individuels. Un modèle ML ne travaille pas dessus directement : on regroupe les paquets en **flux** (même IP source, IP destination, port, protocole), puis on calcule des statistiques sur ce groupe. C'est ce qui permet de détecter du C2 même quand le contenu est chiffré (HTTPS) : on ne regarde jamais le contenu, seulement le **rythme et la forme** du trafic.

### `load_packets()` — lecture du pcap

```python
def load_packets(pcap_file):
    packets = []
    cap = pyshark.FileCapture(pcap_file, display_filter='tcp or udp', only_summaries=False)
    for pkt in cap:
        try:
            proto = pkt.transport_layer
            if proto is None:
                continue
            packets.append({
                'ts': float(pkt.sniff_timestamp),
                'src': pkt.ip.src,
                'dst': pkt.ip.dst,
                'dport': int(pkt[proto].dstport),
                'proto': proto,
                'size': int(pkt.length),
            })
        except AttributeError:
            continue
    cap.close()
    return sorted(packets, key=lambda p: p['ts'])
```

`pyshark` pilote `tshark` en coulisses. Pour chaque paquet TCP/UDP, on extrait seulement 6 métadonnées : horodatage exact, IP source/destination, port, protocole, taille en octets. `display_filter='tcp or udp'` ignore tout le reste (ARP, ICMP, etc.) qui n'est pas pertinent pour caractériser une session applicative.

### Les 12 features et pourquoi chacune compte

```python
def compute_window_features(window_packets, flow_key):
    if len(window_packets) < 4:
        return None
    timestamps = sorted([p['ts'] for p in window_packets])
    sizes = [p['size'] for p in window_packets]
    intervals = np.diff(timestamps)
    ...
```

| Feature | Calcul | Pourquoi elle discrimine C2 / légitime |
|---|---|---|
| `interval_mean` | moyenne des écarts de temps entre paquets | un beacon a un rythme propre (ex: 30s) ; un humain non |
| `interval_std` | écart-type des écarts de temps | **feature la plus discriminante** : un beacon = std très faible (régulier) |
| `interval_cv` | `std / mean` (coefficient de variation) | normalise l'écart-type par l'échelle — permet de comparer un beacon rapide et un beacon lent sur la même échelle |
| `interval_min` / `interval_max` | bornes des écarts | détecte les beacons avec jitter (écart mean/max élevé) |
| `size_mean` | taille moyenne des paquets | un beacon répète souvent la même requête → taille stable |
| `size_std` | écart-type des tailles | un beacon = std faible (paquets quasi identiques) ; une page web varie énormément |
| `size_min` / `size_max` | bornes de taille | complète `size_std` |
| `pkt_count` | nombre de paquets dans la fenêtre | volume brut, peu discriminant seul |
| `session_duration` | durée de la fenêtre observée | contextuel |
| `bytes_per_second` | débit | un C2 léger a un débit faible et constant |

`if len(window_packets) < 4: return None` — on exige au moins 4 paquets pour calculer un écart-type qui ait un sens statistique minimal ; en dessous, la fenêtre est ignorée.

### Le découpage en fenêtres glissantes

```python
def extract_windowed_features(pcap_file, window_seconds=20, step_seconds=10, label=None, source_file=None):
    packets = load_packets(pcap_file)
    t_start = packets[0]['ts']
    t_end = packets[-1]['ts']

    records = []
    window_start = t_start
    while window_start < t_end:
        window_end = window_start + window_seconds
        window_packets = [p for p in packets if window_start <= p['ts'] < window_end]
        flows = defaultdict(list)
        for p in window_packets:
            key = (p['src'], p['dst'], p['dport'], p['proto'])
            flows[key].append(p)
        for flow_key, flow_packets in flows.items():
            feats = compute_window_features(flow_packets, flow_key)
            ...
        window_start += step_seconds
    return pd.DataFrame(records)
```

**Problème résolu par ce découpage** : nos captures sont courtes (quelques minutes) et chacune représente une seule conversation continue. Sans découpage, chaque fichier `.pcap` donnerait **une seule ligne** de dataset (1 flux = 1 ligne), soit 4 lignes au total — bien trop peu pour entraîner quoi que ce soit.

En découpant le temps en **fenêtres de 20 secondes avançant par pas de 10 secondes** (donc chevauchement de 50%), chaque capture produit plusieurs fenêtres, donc plusieurs lignes. C'est la même logique que `detect_live.py` utilisera en conditions réelles (il analyse le trafic par tranches de temps, pas paquet par paquet).

---

## 2. Construction du dataset — `build_dataset_windowed.py`

```python
FILES = [
    ('c2_regular.pcap', 1),
    ('c2_jitter.pcap', 1),
    ('c2_lent.pcap', 1),
    ('legit_browsing.pcap', 0),
]
WINDOW_SECONDS = 20
STEP_SECONDS = 10
```

C'est ici qu'on fait de l'**apprentissage supervisé** : chaque fichier reçoit manuellement son label (`1` = trafic C2, `0` = trafic légitime), car on sait à l'avance ce qu'on a capturé (on a nous-même lancé les beacons Sliver ou la navigation web). Le modèle apprendra ensuite à retrouver ce label à partir des seules 12 features, sans qu'on lui dise explicitement la règle.

Le script appelle `extract_windowed_features()` pour chaque fichier, empile (`pd.concat`) tous les résultats, puis sauvegarde en CSV.

### Résultat obtenu

```
c2_regular.pcap (C2)        -> 32 échantillons
c2_jitter.pcap (C2)         -> 44 échantillons
c2_lent.pcap (C2)           -> 25 échantillons
legit_browsing.pcap (légitime) -> 40 échantillons

Total : 141 échantillons (101 C2, 40 légitime)
```

---

## 3. Entraînement du modèle — `train_model.py`

### Séparation train/test

```python
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.25, random_state=42, stratify=y
)
```

Étape non négociable en ML : on coupe le dataset en deux parties **qui ne se chevauchent jamais**.
- **75% (train)** : le modèle apprend les patterns sur ces exemples.
- **25% (test)** : mis de côté, jamais montré pendant l'apprentissage — sert uniquement à vérifier si le modèle généralise, plutôt que d'avoir simplement mémorisé les exemples d'entraînement (sur-apprentissage / overfitting).

`stratify=y` garantit que la proportion C2/légitime est respectée dans les deux morceaux. `random_state=42` fixe la graine aléatoire pour que le découpage soit reproductible (même résultat à chaque exécution).

Avec 141 lignes, le test set final fait `141 × 0.25 ≈ 36` lignes — c'est très peu, ce qui explique en partie les scores parfaits obtenus (voir section "Limites" plus bas).

### Pourquoi ces 3 modèles précisément

| Modèle | Principe | Pourquoi il est dans la comparaison |
|---|---|---|
| **Régression Logistique** | Trace une frontière linéaire (pondération des 12 features) qui sépare les deux classes | Sert de **baseline interprétable** : si ce modèle simple suffit, pas besoin de complexité inutile. Les coefficients appris sont directement lisibles (quelle feature pèse dans quel sens). |
| **Random Forest** | Construit 200 arbres de décision indépendants, chacun pose une série de questions sur les features, et on fait voter tous les arbres | Capture des relations **non-linéaires** entre features (ex: "C2 seulement si interval_cv faible ET size_std faible simultanément") que la régression logistique simple ne peut pas représenter. Robuste au bruit, peu sensible aux valeurs aberrantes. |
| **XGBoost** | Variante d'arbres de décision où chaque nouvel arbre corrige spécifiquement les erreurs des arbres précédents (boosting) | Souvent le plus performant en pratique sur des données tabulaires de ce type, au prix d'un risque légèrement plus élevé de sur-apprentissage sur petit dataset. |

L'idée n'est pas de deviner à l'avance lequel est le meilleur, mais de **les comparer systématiquement** sur les mêmes données et garder le plus performant — c'est ce que fait la fonction `evaluate()` pour chacun.

### Paramètres importants

```python
LogisticRegression(max_iter=1000, class_weight='balanced')
RandomForestClassifier(n_estimators=200, max_depth=10, class_weight='balanced', random_state=42)
XGBClassifier(n_estimators=200, max_depth=6, learning_rate=0.1, eval_metric='logloss', random_state=42)
```

- **`class_weight='balanced'`** : compense le déséquilibre 101 C2 vs 40 légitime. Sans ça, un modèle pourrait "tricher" en favorisant la classe majoritaire (C2) et obtenir un score brut flatteur sans vraiment apprendre à distinguer les deux.
- **`max_iter=1000`** (LogReg) : nombre d'itérations pour que l'algorithme d'optimisation converge ; valeur par défaut (100) parfois insuffisante.
- **`n_estimators=200`** (RF, XGB) : nombre d'arbres construits — plus d'arbres = généralement plus stable, au prix du temps de calcul (négligeable ici avec 141 lignes).
- **`max_depth`** : profondeur maximale de chaque arbre — limite la complexité que chaque arbre peut apprendre, ce qui réduit le risque de sur-apprentissage sur un petit dataset.
- **`StandardScaler`** (utilisé uniquement pour la régression logistique) : normalise chaque feature pour qu'elle ait une moyenne de 0 et un écart-type de 1. Indispensable ici car `bytes_per_second` (valeurs de l'ordre de 10 000-50 000) et `interval_cv` (valeurs entre 0 et 4) ont des échelles totalement différentes — sans normalisation, la régression logistique donnerait artificiellement plus de poids à la feature avec les plus grandes valeurs brutes. Random Forest et XGBoost n'en ont pas besoin (ils comparent des seuils par feature, indépendamment de l'échelle).

### Les métriques expliquées

```python
f1 = f1_score(y_test, y_pred)
auc = roc_auc_score(y_test, y_proba)
print(classification_report(y_test, y_pred, target_names=['legit', 'C2']))
print(confusion_matrix(y_test, y_pred))
```

**Matrice de confusion** — tableau 2×2 qui croise la vérité et la prédiction :

```
                  Prédit légitime    Prédit C2
Vraiment légitime      VN                FP
Vraiment C2            FN                VP
```

- VP (Vrai Positif) : C2 correctement détecté
- VN (Vrai Négatif) : légitime correctement laissé passer
- FP (Faux Positif) : légitime classé à tort comme C2 → fausse alerte, fatigue de l'analyste SOC
- FN (Faux Négatif) : C2 raté → le pire cas en sécurité, l'attaque passe inaperçue

**Précision** (`precision = VP / (VP + FP)`) : quand le modèle dit "C2", a-t-il raison ? Une précision basse = beaucoup de fausses alertes.

**Rappel** (`recall = VP / (VP + FN)`) : parmi tout le vrai C2 présent, combien le modèle en a-t-il trouvé ? Un rappel bas = des attaques passent inaperçues.

**F1-score** (`2 × précision × rappel / (précision + rappel)`) : moyenne harmonique des deux — pénalise fortement si l'un des deux est très bas, même si l'autre est excellent. C'est la métrique utilisée ici pour départager les 3 modèles, car en détection d'intrusion on veut un bon équilibre (ni trop d'alertes inutiles, ni d'attaques ratées), pas juste optimiser un seul des deux côtés.

**ROC-AUC** : mesure la capacité à bien classer sur **tous les seuils de décision possibles** (pas seulement à 50% de confiance) — utile pour juger la qualité de `predict_proba()` indépendamment du seuil choisi ensuite pour déclencher une alerte.

### Résultat obtenu sur ce dataset

```
=== Régression Logistique ===
              precision    recall  f1-score   support
       legit       1.00      1.00      1.00        10
          C2       1.00      1.00      1.00        26
F1-score : 1.0000 | ROC-AUC : 1.0000

=== Random Forest ===     F1-score : 1.0000 | ROC-AUC : 1.0000
=== XGBoost ===            F1-score : 1.0000 | ROC-AUC : 1.0000

Meilleur modèle : Régression Logistique (F1=1.0000, AUC=1.0000)
```

Les 3 modèles obtiennent un score parfait. **Ce n'est pas un signe que le problème est "résolu"** — voir la section Limites ci-dessous pour l'interprétation correcte de ce résultat.

En cas d'égalité parfaite entre les 3 modèles (comme ici), `train_model.py` garde le premier testé qui atteint le score maximal — la Régression Logistique. C'est en réalité un choix favorable pour un POC de thèse : c'est le modèle le plus simple et le plus interprétable des trois (on peut directement lire quels coefficients pèsent dans la décision), contrairement à Random Forest/XGBoost qui sont plus "boîte noire".

### Pourquoi le score est parfait ici (et pourquoi s'en méfier)

```python
print(df.groupby('label')['size_std'].describe())
# label 0 (légitime) : size_std = 0.0 sur toute la distribution
# label 1 (C2)       : size_std moyenne ≈ 603, écart-type ≈ 188
```

En creusant les données, `size_std` (écart-type de la taille des paquets) vaut **exactement 0** sur tout le trafic légitime capturé, alors qu'elle varie largement sur le trafic C2. C'est probablement un artefact de la méthode de capture du trafic légitime (`curl -s -o /dev/null` génère des requêtes très uniformes, pas une vraie navigation avec des pages de tailles variées) plutôt qu'une propriété générale du trafic légitime. Le modèle a donc pu apprendre une séparation quasi triviale sur cette seule feature, ce qui explique le F1 = 1.0 sur un dataset aussi petit et homogène.

De même, `interval_cv` (coefficient de variation des intervalles) est nettement plus élevé en moyenne sur le C2 (≈1.29) que sur le légitime (≈0.47) dans cet échantillon — c'est cohérent avec la théorie (un beacon devrait être plus régulier, donc `interval_cv` plus bas), mais ici c'est l'inverse qui apparaît, probablement parce que le trafic Sliver inclut des polls HTTP irréguliers en plus du rythme de beaconing de fond, et parce que les flux retour PC→Pi3 sont mélangés aux flux Pi3→PC. **Ce résultat contre-intuitif est un signal qu'il faudra affiner l'extraction de features (séparer les flux par direction, isoler le pattern de beaconing du bruit HTTP) avant de tirer des conclusions pour la thèse.**

### Sauvegarde du modèle

```python
if best['scaler'] is not None:
    joblib.dump({'model': best['model'], 'scaler': best['scaler']}, args.out)
else:
    joblib.dump(best['model'], args.out)
```

Le fichier `.joblib` contient soit le modèle seul, soit un dictionnaire `{model, scaler}` si le modèle gagnant a besoin de features normalisées (cas de la Régression Logistique ici) — c'est pour ça que le code qui charge le modèle plus tard doit vérifier ce cas (`isinstance(obj, dict)`).

---

## 4. De la prédiction à l'alerte — `init_db.py` / `app.py`

```python
def load_model(path):
    obj = joblib.load(path)
    if isinstance(obj, dict):
        return obj['model'], obj.get('scaler')
    return obj, None

def predict(model, scaler, X):
    X_in = scaler.transform(X) if scaler is not None else X
    preds = model.predict(X_in)
    probas = model.predict_proba(X_in)[:, 1]
    return preds, probas
```

`model.predict(X)` donne une réponse binaire (0 ou 1). `model.predict_proba(X)[:, 1]` donne la **probabilité estimée** que l'échantillon soit de classe 1 (C2) — c'est un nombre continu entre 0 et 1, beaucoup plus riche qu'un simple oui/non : on distingue "je pense que c'est du C2 mais pas sûr" (0.55) de "quasi certain" (0.97).

### La règle qui transforme la probabilité en alerte lisible

```python
def severity_from_confidence(label, confidence):
    if label == 0:
        return 'info'
    if confidence >= 0.85:
        return 'critical'
    elif confidence >= 0.5:
        return 'warning'
    return 'info'
```

C'est cette fonction qui fait le pont entre la sortie mathématique du modèle et ce qu'un analyste SOC voit dans le dashboard :
- `label == 0` (le modèle classe en légitime) → **vert**, quelle que soit la confiance
- `label == 1` (C2) avec confiance ≥ 0.85 → **rouge** (critique, à traiter immédiatement)
- `label == 1` avec confiance entre 0.5 et 0.85 → **orange** (warning, à surveiller)

Ces seuils (0.5 et 0.85) sont des choix arbitraires raisonnables pour un POC, pas des valeurs calculées depuis les données — à ajuster une fois qu'on aura un vrai dataset de validation représentatif (par exemple en traçant la courbe précision/rappel en fonction du seuil, et en choisissant le point qui correspond au compromis souhaité entre fausses alertes et attaques ratées).

### Alertes email — logique de déclenchement

```python
ALERT_THRESHOLD_CRITICAL = 3
ALERT_WINDOW_MINUTES = 10
ALERT_COOLDOWN_SECONDS = 600

def check_and_send_alert():
    if now - _last_alert_sent['ts'] < ALERT_COOLDOWN_SECONDS:
        return
    count = conn.execute(f"""
        SELECT COUNT(*) FROM logs
        WHERE severity = 'critical'
        AND timestamp >= datetime('now', '-{ALERT_WINDOW_MINUTES} minutes')
    """).fetchone()['c']
    if count >= ALERT_THRESHOLD_CRITICAL:
        send_alert_email(count)
```

Un email n'est envoyé que si **3 logs critiques ou plus** apparaissent dans une fenêtre de **10 minutes** — évite de déclencher une alerte sur un seul flux isolé qui pourrait être un faux positif. Le `cooldown` de 10 minutes empêche d'envoyer un email à chaque nouvelle détection une fois le seuil dépassé (sinon spam continu pendant une vraie attaque en cours).

---

## 5. Le dashboard — sécurité et architecture

### Pourquoi les requêtes SQL libres sont sécurisées

```python
FORBIDDEN_KEYWORDS = re.compile(
    r'\b(INSERT|UPDATE|DELETE|DROP|ALTER|CREATE|ATTACH|PRAGMA|REPLACE|TRUNCATE)\b',
    re.IGNORECASE
)

def is_safe_select(query):
    q = query.strip().rstrip(';')
    if not q.upper().startswith('SELECT'):
        return False, "Seules les requêtes SELECT sont autorisées."
    if FORBIDDEN_KEYWORDS.search(q):
        return False, "Mot-clé non autorisé détecté dans la requête."
    if ';' in q:
        return False, "Les requêtes multiples ne sont pas autorisées."
    return True, None
```

Deux niveaux de protection, testés et validés :
1. **Filtre textuel** : la requête doit commencer par `SELECT`, ne doit contenir aucun mot-clé de modification (`DROP`, `DELETE`...), et ne doit pas chaîner plusieurs requêtes avec `;`.
2. **Connexion SQLite en lecture seule** au niveau du système de fichiers (`sqlite3.connect(f'file:{DB_PATH}?mode=ro', uri=True)`) — même si une requête malveillante parvenait à passer le filtre textuel, la base refuserait physiquement toute écriture.

Vérifié en conditions réelles : `DROP TABLE logs` → rejeté par le filtre. `SELECT 1; DROP TABLE logs;` → rejeté (mot-clé + requêtes multiples détectés).

---

## Limites actuelles et prochaines étapes pour la thèse

1. **Dataset trop petit et trop homogène** : 141 lignes issues de seulement 4 captures courtes. Le score parfait (F1=1.0) reflète la facilité de séparation sur cet échantillon précis, pas une performance généralisable. Pour des résultats exploitables dans la thèse, il faudra refaire des captures **plus longues et plus variées** (plusieurs sessions de beacon indépendantes par scénario, plusieurs sites web réels visités avec un vrai navigateur plutôt que `curl -o /dev/null`).
2. **`size_std` = 0 sur tout le trafic légitime** : probable artefact de la méthode de capture (curl en mode silencieux), pas une propriété générale du trafic web normal — à corriger en variant les sources de trafic légitime (navigateur réel, plusieurs types de contenu : images, vidéos, API).
3. **`interval_cv` plus élevé sur le C2 que le légitime dans cet échantillon**, contraire à l'intuition théorique (un beacon devrait être plus régulier) — à investiguer en séparant les flux par direction (Pi3→PC vs PC→Pi3) et en isolant le pattern de beaconing du bruit des requêtes HTTP/polling.
4. **Seuils de sévérité (0.5 / 0.85) choisis arbitrairement** — à recalibrer une fois un dataset de validation représentatif disponible.
5. **Pas d'authentification sur le dashboard** — acceptable pour un POC local, à ajouter avant toute exposition réseau plus large.
