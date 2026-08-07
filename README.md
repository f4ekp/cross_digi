# APRS Cross-Digipeater — RF 144.800 MHz ↔ LoRa APRS

Ce projet met en place un **cross-digipeater APRS** (ou « pont VHF ↔ LoRa ») sur Raspberry Pi, faisant le lien entre :

- Le réseau APRS radio classique **144.800 MHz (AFSK 1200 bauds)**, géré nativement par **[Direwolf](https://github.com/wb2osz/direwolf)**
- Le réseau **LoRa APRS** (généralement 433.775 MHz), géré par un module exécutant le firmware **[iGate/digipeater de CA2RXU](https://github.com/richonguzman)** (Ricardo Guzman)

Ce montage s'appuie **directement sur les fonctionnalités `NCHANNEL` / `SCHANNEL` de Direwolf** (introduites en version 1.8 pour le réseau, 1.9 pour le port série), qui permettent de déclarer un canal virtuel connecté à un TNC KISS externe — via TCP/IP (Wi-Fi/Ethernet) ou via port série (USB). Le module LoRa CA2RXU, qui expose lui-même une interface KISS, est ainsi vu par Direwolf comme un canal radio supplémentaire. Direwolf réalise alors le cross-digipeat entre son canal radio réel (144.800 MHz) et ce canal LoRa, via sa configuration `DIGIPEAT` habituelle — **sans script ni processus tiers**.

📄 Référence officielle : *Lora APRS - VHF APRS Bridge*, John Langner (WB2OSZ), Direwolf — [APRS-LoRa-VHF-APRS-Bridge.pdf](https://github.com/wb2osz/direwolf-doc/blob/main/APRS-LoRa-VHF-APRS-Bridge.pdf)
(méthode initialement défrichée par Geoffrey, F4FXL — [Building a VHF / LoRa APRS Bridge](https://www.f4fxl.org/building-a-vhf-lora-aprs-bridge/))

L'objectif est de permettre aux stations APRS classiques (144.800 MHz) et aux stations LoRa APRS de se voir mutuellement, en relayant les trames d'un réseau vers l'autre.

Ce dépôt documente **les deux méthodes de liaison possibles** entre Direwolf et le module LoRa CA2RXU :
- **`NCHANNEL`** — liaison réseau (Wi-Fi/Ethernet), le module LoRa expose son KISS en TCP
- **`SCHANNEL`** — liaison série (USB), le module LoRa expose son KISS sur port série

---

## 🏗️ Architecture

```
             ┌──────────────┐
             │  2m radio     │
             │  144.800 MHz  │
             └───────┬───────┘
                      │  (Channel 0, interne)
                      ▼
   ┌──────────────────────────────────┐
   │            Direwolf                │
   │   TNC logiciel + digipeater + IGate │
   └──────────────────────┬────────────┘
                      ▲    │  (Channel 11, externe)
                      │    ▼
             ┌────────┴──────────┐
             │  LoRa APRS TNC     │
             │  (firmware CA2RXU) │
             │  KISS via NCHANNEL │
             │  ou SCHANNEL       │
             └────────────────────┘
```

Direwolf gère un canal radio interne classique (**CHANNEL 0**, la VHF 144.800 MHz) et un canal externe (numéroté par exemple **11**) qui pointe vers le module LoRa, soit en réseau (`NCHANNEL`), soit en série (`SCHANNEL`). Ce canal externe est ensuite traité exactement comme un canal radio normal : digipeat, IGate, beacon, filtrage, etc.

---

## 📦 Matériel utilisé

- Raspberry Pi (modèle : *à préciser*)
- Interface radio pour Direwolf (carte son USB + radio VHF 144.800 MHz, ou modem TNC dédié)
- Module LoRa APRS compatible firmware CA2RXU (ex. LilyGo T-Beam, Heltec, etc. — *à préciser selon ton matériel*), en mode IGate/Digipeater
- Antennes VHF (144.800 MHz) et LoRa (433 MHz, 433.775 MHz par défaut en Europe)

---

## 🖥️ Logiciels utilisés

| Composant | Rôle | Lien |
|---|---|---|
| [Direwolf](https://github.com/wb2osz/direwolf) | TNC logiciel AFSK 1200 bds sur 144.800 MHz **+** cross-digipeat vers le canal LoRa (`NCHANNEL`/`SCHANNEL`) | github.com/wb2osz/direwolf |
| Firmware **iGate/digipeater CA2RXU** | Gestion LoRa APRS, expose une interface KISS (série et/ou TCP) | github.com/richonguzman |

---

## ⚙️ Installation

### 1. Flasher et configurer le module LoRa (firmware CA2RXU)

- Flasher le firmware iGate/digipeater CA2RXU sur le module (voir sa documentation)
- Configurer l'indicatif, la fréquence LoRa (433.775 MHz par défaut en Europe) et les paramètres LoRa (SF, BW, CR)
- **Ne pas activer** les fonctions digipeater/IGate internes du module : il doit être utilisé en simple TNC KISS, tout le digipeat/IGate étant délégué à Direwolf
- Dans l'écran **TNC** du firmware, selon la méthode choisie :

  **Pour une liaison réseau (`NCHANNEL`)** :
  - Activer **« Enable TNC server (Port 8001) »**
  - Activer **« Accept own frames via KISS »**
  - Relever l'**adresse IP** du module sur le LAN (Wi-Fi), ex. `192.168.1.238`

  **Pour une liaison série (`SCHANNEL`, nécessite Direwolf ≥ 1.9)** :
  - Activer **« Enable Serial KISS »**
  - Activer **« Accept own frames via KISS »**
  - Relever le **port série** utilisé une fois le module branché en USB (ex. `/dev/ttyACM0` sous Linux)

### 2. Installer Direwolf

```bash
sudo apt update
sudo apt install direwolf
```

> ℹ️ `SCHANNEL` (port série) nécessite Direwolf 1.9 (branche « dev » au moment de la rédaction de la doc officielle). `NCHANNEL` (réseau) est disponible depuis la 1.8. Vérifie ta version avec `direwolf --version` et recompile depuis les sources si besoin.

### 3. Configurer `direwolf.conf`

Structure commune : un **canal radio interne** (0) pour la VHF, et un **canal externe** (ici 11) pour le LoRa, via `NCHANNEL` **ou** `SCHANNEL` selon la méthode retenue.

```
MYCALL <TON_INDICATIF>-<SSID>

# --- Canal 0 : radio VHF 144.800 MHz (AFSK 1200 bauds, TNC interne) ---
ADEVICE plughw:1,0
CHANNEL 0
MODEM 1200
PTT GPIO 23

# --- Canal 11 : module LoRa CA2RXU, via KISS ---
# Choisir UNE des deux lignes ci-dessous selon la méthode de liaison :

# Option A - liaison réseau (Wi-Fi/Ethernet)
NCHANNEL 11 192.168.1.238 8001

# Option B - liaison série (USB)
#SCHANNEL 11 /dev/ttyACM0 115200

# --- Cross-digipeat entre VHF (0) et LoRa (11), dans les deux sens ---
DIGIPEAT 0 11 ^WIDE[3-7]-[1-7]$|^TEST$ ^WIDE[12]-[12]$
DIGIPEAT 11 0 ^WIDE[3-7]-[1-7]$|^TEST$ ^WIDE[12]-[12]$

# --- (optionnel) digipeat LoRa -> LoRa, pour profiter des fonctions avancées de Direwolf ---
#DIGIPEAT 11 11 ^WIDE[3-7]-[1-7]$|^TEST$ ^WIDE[12]-[12]$
```

> ℹ️ Remplace `192.168.1.238`/`8001` (option réseau) ou `/dev/ttyACM0`/`115200` (option série) par les valeurs réelles de ton module. Le nom d'hôte peut aussi être utilisé à la place de l'IP si ton routeur le supporte (ex. `NCHANNEL 11 iGATE-<INDICATIF>-11 8001`).

Lancement :
```bash
direwolf -c direwolf.conf -t 0
```

Au démarrage, Direwolf affiche les canaux configurés, par exemple :
```
Channel 0: 1200 baud, AFSK 1200 & 2200 Hz, A+, 44100 sample rate.
Channel 11: Network TNC 192.168.1.238 8001
```

### 4. Tester la liaison KISS avant d'activer le digipeat

Avant d'ajouter les lignes `DIGIPEAT`, on peut vérifier que le canal LoRa reçoit bien des trames avec l'utilitaire `kissutil`, fourni avec Direwolf :

```bash
# Liaison réseau
kissutil -h 192.168.1.238

# Liaison série
kissutil -p /dev/ttyACM0 -s 115200
```

Une trame reçue d'un tracker LoRa s'affiche ainsi :
```
[0] WB2OSZ-12>APLRT1,WIDE1-1:=/8wPx<K"SbV$G Bat=4.17V (100%)
```

---

## 🔁 Fonctionnement

1. Direwolf démodule/module l'AFSK 1200 bauds sur le **canal 0** (radio VHF 144.800 MHz)
2. Il communique avec le module LoRa CA2RXU sur le **canal 11**, via KISS réseau (`NCHANNEL`) ou série (`SCHANNEL`) selon la config choisie — ce canal est ensuite traité comme un canal radio ordinaire (digipeat, IGate, beacon, filtrage...)
3. Les règles **`DIGIPEAT`** appliquent la logique de digipeat standard (alias type `WIDE1-1`, `WIDE2-1`, etc.) entre les deux canaux, dans les deux sens
4. Chaque trame reçue sur un réseau et correspondant aux critères de digipeat est automatiquement réémise sur l'autre, avec mise à jour du chemin AX.25 — par exemple, une trame reçue en LoRa (`[11]`) et relayée vers la VHF apparaît comme `[0L]` dans les logs Direwolf
5. Des directives **`FILTER`** optionnelles permettent d'affiner précisément ce qui est relayé entre chaque paire de canaux (ex. `FILTER 0 11 ...`, `FILTER 11 0 ...`), y compris pour contrôler ce qui remonte vers **APRS-IS** via l'IGate (ex. `FILTER 11 IG 0` pour empêcher les trames LoRa de remonter vers APRS-IS)

Aucun script ni processus tiers n'est nécessaire : tout est géré nativement par Direwolf, qui peut aussi assurer la fonction d'IGate pour le réseau LoRa.

---

## 🗺️ Visualisation

Une application d'affichage APRS classique (APRSISCE/32, YAAC, PinPoint APRS, Xastir...) configurée pour utiliser Direwolf comme TNC réseau affichera sur une même carte les stations VHF **et** LoRa.

---

## 🚀 Démarrage automatique (systemd)

Exemple de service systemd pour lancer Direwolf au démarrage du Raspberry Pi :

```ini
[Unit]
Description=Direwolf APRS cross-digipeater (VHF <-> LoRa)
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/bin/direwolf -c /home/pi/direwolf.conf -t 0
Restart=on-failure
User=pi

[Install]
WantedBy=multi-user.target
```

```bash
sudo cp direwolf-cross-digi.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now direwolf-cross-digi.service
```

> ℹ️ Si tu utilises la liaison série (`SCHANNEL`), pense à ordonner le service pour qu'il démarre après la détection du périphérique USB (ex. règle udev ou `After=dev-ttyACM0.device`).

---

## 📄 Licence

*(à préciser — MIT, GPL, etc.)*

---

## 🙏 Remerciements

- [John Langner, WB2OSZ](https://github.com/wb2osz) pour Direwolf et la documentation du pont VHF/LoRa
- [Geoffrey, F4FXL](https://www.f4fxl.org/) pour les travaux précurseurs sur le pont VHF ↔ LoRa APRS
- [Ricardo Guzman, CA2RXU](https://github.com/richonguzman) pour le firmware iGate/digipeater LoRa APRS
- La communauté APRS/LoRa APRS

---

## 📡 Contact

*(ton indicatif radioamateur / contact)*
