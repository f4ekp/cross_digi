# 🎛️ GPIO Control via APRS

> Pilotez les sorties GPIO de votre Raspberry Pi à distance, par radio, en envoyant un simple message APRS.

![Raspberry Pi](https://img.shields.io/badge/Raspberry%20Pi-BCM-red?logo=raspberry-pi)
![Python](https://img.shields.io/badge/Python-3.11%2B-blue?logo=python)
![Direwolf](https://img.shields.io/badge/TNC-Direwolf-green)
![Licence](https://img.shields.io/badge/licence-MIT-lightgrey)

---

## 📖 Présentation

Ce projet permet de **contrôler des sorties GPIO d'un Raspberry Pi à distance**, en lui envoyant un message APRS depuis n'importe quel poste radio équipé d'un TNC ou d'un logiciel compatible (APRSdroid, APRS.fi, Xastir…).

Le Raspberry Pi tourne en digipeateu avec **Direwolf** comme TNC logiciel. Un script Python écoute les paquets entrants et exécute les commandes GPIO en temps réel, avec accusé de réception APRS automatique.

### Cas d'usage typiques

- 🔌 Allumer/éteindre un relais à distance
- 💡 Contrôler un éclairage depuis le terrain
- 🔔 Déclencher une alarme ou une sirène
- 🚪 Commander une gâche électrique
- 📡 Basculer une antenne ou un amplificateur

---

## 🗺️ Architecture

```
[ Poste radio distant ]
        │  message APRS : F4EKP>APDR16::F4EKP-3:!out$ 17{5
        │
       RF
        │
[ Raspberry Pi ]
  ┌─────────────────────────────────────┐
  │  Carte son / Interface radio        │
  │           ↓                         │
  │       Direwolf (TNC)                │
  │       KISS TCP :8001                │
  │           ↓                         │
  │    gpio_control.py                  │
  │    ┌──────────────────────┐         │
  │    │ Décode le paquet     │         │
  │    │ Vérifie le callsign  │         │
  │    │ Exécute sur GPIO BCM │         │
  │    │ Renvoie un ACK APRS  │         │
  │    └──────────────────────┘         │
  │           ↓                         │
  │    [ Broches GPIO ]                 │
  │    GPIO 17 → relais, LED…           │
  └─────────────────────────────────────┘
```

---

## ⚙️ Prérequis

### Matériel

| Élément | Détail |
|---|---|
| Raspberry Pi | N'importe quel modèle avec GPIO 40 broches |
| Interface radio | Carte son USB + câble PTT, ou interface dédiée (SignaLink, Digirig…) |
| Émetteur-récepteur | VHF/UHF sur 144.800 MHz (fréquence APRS Europe) |

### Logiciel

- Raspberry Pi OS (Bullseye ou Bookworm)
- Python 3.11+
- **Direwolf** déjà installé et configuré

> 💡 Ce guide suppose que **Direwolf est déjà fonctionnel** sur votre Raspberry Pi. Si ce n'est pas le cas, consultez d'abord la [documentation officielle de Direwolf](https://github.com/wb2osz/direwolf).

---

## 📁 Fichiers du projet

```
gpio_aprs/
├── gpio_control.py       ← Script principal
├── config.ini            ← Configuration
└── gpio-aprs.service     ← Service systemd (démarrage automatique)
```

---

## 🚀 Installation pas à pas

### Étape 1 — Configurer Direwolf

Ouvrez votre fichier de configuration Direwolf (généralement `/etc/direwolf.conf` ou `/home/pi/direwolf.conf`) et vérifiez la présence de cette ligne :

```
KISSPORT 8001
```

Si elle est absente, ajoutez-la et redémarrez Direwolf.

> ⚠️ Sans `KISSPORT 8001`, le script ne pourra pas communiquer avec Direwolf.

---

### Étape 2 — Télécharger les fichiers

```bash
# Créer le dossier de travail
mkdir -p /home/f4ekp/gpio_aprs
cd /home/f4ekp/gpio_aprs

# Télécharger les fichiers (ou les copier manuellement)
# gpio_control.py, config.ini, gpio-aprs.service
```

---

### Étape 3 — Installer la dépendance Python

```bash
pip3 install RPi.GPIO --break-system-packages
# ou selon votre système :
sudo apt install python3-rpi.gpio
```

---

### Étape 4 — Créer le fichier de log

```bash
sudo touch /var/log/gpio_aprs.log
sudo chown f4ekp:f4ekp /var/log/gpio_aprs.log
```

---

### Étape 5 — Configurer le script

Éditez `config.ini` avec votre indicatif et vos GPIO :

```bash
nano /home/f4ekp/gpio_aprs/config.ini
```

```ini
[gpio_aprs]

; Connexion à Direwolf
kiss_host = 127.0.0.1
kiss_port = 8001
reconnect_delay = 10

; Votre indicatif (indispensable pour envoyer les ACK)
mycall = F4EKP-3

; Indicatifs autorisés à envoyer des commandes
; Laisser vide pour tout accepter (déconseillé)
allowed_calls = F4EKP

; Broches GPIO BCM autorisées (voir tableau plus bas)
allowed_gpios = 17,27,22,5,6,13,19,26

; Journalisation
log_file = /var/log/gpio_aprs.log
log_level = INFO
log_max_bytes = 1048576
log_backup_count = 3
```

> 🔒 **Sécurité** : renseignez toujours `allowed_calls` avec votre propre indicatif. Laisser ce champ vide permet à n'importe qui d'entendre votre digi de contrôler vos GPIO.

---

### Étape 6 — Tester manuellement

Avant d'installer le service, testez le script directement dans le terminal :

```bash
cd /home/f4ekp/gpio_aprs
python3 gpio_control.py --config config.ini
```

Vous devez voir :

```
2026-01-01 12:00:00 [INFO] === gpio_control démarré ===
2026-01-01 12:00:00 [INFO] mycall : F4EKP-3
2026-01-01 12:00:00 [INFO] GPIO autorisés : [17, 27, 22, 5, 6, 13, 19, 26]
2026-01-01 12:00:00 [INFO] Connecté.
```

Envoyez une commande test depuis votre logiciel APRS. Si tout fonctionne, passez à l'étape suivante. Arrêtez le script avec `Ctrl+C`.

---

### Étape 7 — Installer le service systemd

Le service systemd permet au script de **démarrer automatiquement** au boot et de **redémarrer tout seul** en cas de problème.

```bash
# Copier le fichier service
sudo cp /home/f4ekp/gpio_aprs/gpio-aprs.service /etc/systemd/system/

# Activer et démarrer le service
sudo systemctl daemon-reload
sudo systemctl enable gpio-aprs
sudo systemctl start gpio-aprs

# Vérifier que tout est OK
sudo systemctl status gpio-aprs
```

Vous devez voir `● gpio-aprs.service` avec le statut **`active (running)`** en vert.

---

## 📻 Envoyer une commande depuis le terrain

### Syntaxe du message APRS

Depuis votre logiciel APRS, envoyez un **message** (pas un commentaire de position) à destination de votre digi, avec ce texte :

| Commande | Effet |
|---|---|
| `!out$ 17` | GPIO 17 → **HIGH** (mise à 1) |
| `!out$ 17=1` | GPIO 17 → **HIGH** |
| `!out$ 17=0` | GPIO 17 → **LOW** (mise à 0) |
| `!out$ 17=T` | GPIO 17 → **bascule** (toggle) |

> 💡 La commande peut apparaître n'importe où dans le texte du message. `Allume la lampe !out$ 17=1 merci` fonctionnerait très bien.

### Exemple de trame APRS générée par votre logiciel

```
F4EKP>APDR16,WIDE2-2::F4EKP-3   :!out$ 17=1{5
```

- `F4EKP` → votre indicatif (expéditeur)
- `F4EKP-3` → indicatif du digi cible (destinataire)
- `!out$ 17=1` → commande GPIO
- `{5` → numéro de message (géré automatiquement par votre logiciel)

### ACK automatique

Dès que la commande est exécutée, le digi renvoie automatiquement un accusé de réception :

```
F4EKP-3>APRS::F4EKP    :ack5
```

Votre logiciel APRS arrête alors les retransmissions. Sans cet ACK, la plupart des logiciels renvoient la commande plusieurs fois.

---

## 🔌 Numérotation des broches GPIO

Le script utilise la numérotation **BCM** (Broadcom) — c'est la référence standard sur Raspberry Pi.

```
Connecteur 40 broches (vue de dessus, port USB en bas)

 3V3  (1) (2)  5V
GPIO2 (3) (4)  5V
GPIO3 (5) (6)  GND
GPIO4 (7) (8)  GPIO14
 GND  (9) (10) GPIO15
GPIO17(11) (12) GPIO18   ← BCM 17 = broche physique 11
GPIO27(13) (14) GND      ← BCM 27 = broche physique 13
GPIO22(15) (16) GPIO23   ← BCM 22 = broche physique 15
 3V3 (17) (18) GPIO24
GPIO10(19) (20) GND
GPIO9 (21) (22) GPIO25
GPIO11(23) (24) GPIO8
 GND (25) (26) GPIO7
GPIO0 (27) (28) GPIO1
GPIO5 (29) (30) GND      ← BCM 5  = broche physique 29
GPIO6 (31) (32) GPIO12   ← BCM 6  = broche physique 31
GPIO13(33) (34) GND      ← BCM 13 = broche physique 33
GPIO19(35) (36) GPIO16   ← BCM 19 = broche physique 35
GPIO26(37) (38) GPIO20   ← BCM 26 = broche physique 37
 GND (39) (40) GPIO21
```

**Tableau de correspondance rapide**

| Broche physique | BCM (à utiliser dans la commande) |
|:---:|:---:|
| 11 | **17** |
| 13 | **27** |
| 15 | **22** |
| 29 | **5** |
| 31 | **6** |
| 33 | **13** |
| 35 | **19** |
| 37 | **26** |

Pour afficher le brochage complet sur le Pi :
```bash
pinout
```

> ⚠️ Les GPIO du Raspberry Pi fonctionnent en **3,3 V**. Ne connectez jamais directement une charge 5 V ou 12 V. Utilisez un transistor, un relais ou un module de conversion de niveau.

---

## 🛠️ Supervision et dépannage

### Commandes utiles

```bash
# État du service
sudo systemctl status gpio-aprs

# Logs en temps réel
sudo journalctl -u gpio-aprs -f

# Logs dans le fichier
tail -f /var/log/gpio_aprs.log

# Redémarrer le service
sudo systemctl restart gpio-aprs

# Arrêter le service
sudo systemctl stop gpio-aprs
```

### Activer les logs détaillés

Pour diagnostiquer un problème, passez temporairement en mode DEBUG dans `config.ini` :

```ini
log_level = DEBUG
```

Puis redémarrez le service. Vous verrez alors **tous** les paquets reçus, même ceux qui ne contiennent pas de commande.

### Tableau de dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| Service `failed` au démarrage | Direwolf non démarré | Démarrer Direwolf en premier, ou ajouter `After=direwolf.service` dans le `.service` |
| `Connexion échouée` en boucle | `KISSPORT 8001` absent de `direwolf.conf` | Ajouter la ligne et redémarrer Direwolf |
| Commande ignorée (pas de log) | Callsign absent de `allowed_calls` | Vérifier l'indicatif exact (avec ou sans SSID) dans les logs |
| GPIO commandé mais pas d'ACK | `mycall` absent ou incorrect dans `config.ini` | Renseigner `mycall` avec l'indicatif complet (ex: `F4EKP-3`) |
| `GPIO non disponible` | Script lancé sur un PC (pas un Pi) | Normal en mode simulation, ignoré sur le Pi réel |
| Le client renvoie la commande en boucle | ACK non reçu | Vérifier `mycall` dans `config.ini` et les logs |

---

## 🔒 Sécurité

| Recommandation | Pourquoi |
|---|---|
| Toujours renseigner `allowed_calls` | Évite qu'une station inconnue commande vos GPIO |
| Limiter `allowed_gpios` aux broches réellement utilisées | Réduit la surface d'attaque |
| Ne pas exposer le port 8001 sur le réseau | KISS TCP n'est pas authentifié |
| Utiliser des relais optocouplés | Isole galvaniquement le Pi des charges externes |

---

## 📜 Licence

MIT — Libre d'utilisation, de modification et de redistribution.

---

## 🤝 Contribuer

Les issues et pull requests sont les bienvenues. Ce projet est développé pour la communauté radioamateur francophone.

**73 de F4EKP** 📡
