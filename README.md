# HEM-JKRS485-JKBT-PZEM017-ADC1-ADC2

Home Energy Management (ESPHome) — évolution de
[HEM_ESPHOME](https://github.com/C00L4LIFE/HEM_ESPHOME) intégrant le JK BMS.
Version courante : voir [`VERSION`](VERSION) et [`CHANGELOG.md`](CHANGELOG.md).

## Ce qui change par rapport à HEM_ESPHOME

| Avant | Maintenant |
|---|---|
| eSmart3 sur RS485 (16/17/4) | **JK BMS sur ce même port, en écoute seule** |
| JK BMS sur un ESP32 séparé (BLE) | JK en **BLE à la demande, pour paramétrer uniquement** |
| Tension batterie : eSmart3 / JBD | **Référence unique : JK (RS485)** |
| Production : eSmart3 + shunt MPPT2 (ADS A0-A1) | **PZEM-017 côté panneaux**, avec calibration |
| Shunt onduleur (ADS A2-A3) | ADC1 / ADC2 **génériques calibrables**, sans rôle |
| Protection batterie → consigne eSmart3 | Retirée (plus d'actionneur) |
| JBD BMS en BLE | **Inchangé** |

## Câblage

| Périphérique | Bus | Pins ESP32 | Notes |
|---|---|---|---|
| JK BMS (JK-PB, v19) | UART1 RS485, 115200 | **RX 16**, DE/RE **4** (forcé BAS), TX 17 non utilisé | MAX485 existant de l'eSmart3. RJ45 « RS485 » du JK : 1 = B, 2 = A, 3 = GND. DIP du JK = **0x00 (maître)** |
| PZEM-017 | UART2 Modbus RTU, 9600 8N2 | **RX 13 / TX 14 / DE-RE 18** | Nouveau MAX485 (ou module auto-direction : retirer `flow_control_pin`). Alimenter le PZEM en **5 V** par son connecteur com. Shunt sur le **négatif panneaux**, avant le régulateur |
| ADS1115 | I2C 0x48 | SDA 21 / SCL 22 | ADC1 = A0-A1, ADC2 = A2-A3 |
| JBD BMS | BLE | — | MAC `jbd_bms_mac_address` |
| JK BMS (paramétrage) | BLE | — | MAC `jk_bms_mac_address` |
| 4 relais (actifs bas) | GPIO | 25, 26, 27, 32 | |
| 4 entrées | GPIO | 33 (pull-up), 34/35/36 (pull-up externe) | |
| LED statut | GPIO | 2 | |

UART : UART0 = logger, UART1 = JK, UART2 = PZEM-017 (les 3 UART matériels de l'ESP32).

### Pourquoi ces pins pour le PZEM-017
13/14 étaient le bus du PZEM-004T (désactivé) : aucun changement de câblage
côté ESP. GPIO18 est libre, n'est pas une broche de démarrage (strapping) et
ne bouge pas au boot : idéal pour le DE/RE.

## JK BMS : écoute RS485 + paramétrage Bluetooth

- **RS485 (mesures)** : le JK maître (adresse 0) publie ses trames sur son bus
  RS485 interne ; l'ESP les écoute. Aucune émission possible : DE/RE maintenu
  à 0 et UART sans TX. Les infos appareil (modèle, versions) ne passent jamais
  sur ce bus → lues en Bluetooth.
- **Bluetooth (paramètres)**, deux modes choisis par **« JK BT Connexion
  auto »** (persistant) :
  - OFF = **à la demande** : **« JK BT Connexion »** pour se connecter,
    coupure auto après **« JK BT Déconnexion auto »** minutes (0 = jamais) ;
  - ON = **automatique** : connexion au démarrage, liaison permanente.
  La liaison est toujours coupée pendant une OTA. Le JK n'accepte qu'une connexion BLE : tant
  que c'est OFF, l'app JK du téléphone est utilisable. Tous les réglages sont
  préfixés « JK Param … ».

## Calibration

### PZEM-017 (production PV) — `valeur = brute × gain/100 + offset`
1. **« PZEM Calibre shunt »** : choisir le shunt réellement installé (50/100/200/300 A) —
   écrit dans le PZEM.
2. **Zéro** : de nuit, bouton **« PV Calib. Zéro courant »**.
3. **Tension** : saisir la mesure du multimètre dans « PV Calib. Référence
   tension » → **« PV Calib. Appliquer réf. tension »**.
4. **Courant** : pince DC en plein soleil → « PV Calib. Référence courant » →
   **« PV Calib. Appliquer réf. courant »**.
5. « PV Calib. Réinitialiser » remet gain 100 % / offset 0.

Les valeurs brutes restent visibles (« PZEM Tension brute », « PZEM Courant
brut », diagnostic) pour contrôle.

### ADC1 / ADC2
« ADCx Shunt A nominal » / « ADCx Shunt mV nominal » (défaut 100 A / 75 mV),
« ADCx Inverser sens », puis les mêmes outils : Zéro, Référence + Appliquer,
gain/offset manuels.

## Home Assistant

Créer une fois le helper **`input_number.hem_jk_pv_energy_total`** (0 à
999999, pas 0.1) et autoriser l'appareil à exécuter des actions HA
(Paramètres → Appareils → ESPHome → hem-jk → Configurer). Sans cela,
l'énergie PV repose uniquement sur la NVS locale.

Tableau de bord Énergie : production solaire = **« PV Énergie totale »** ;
batterie = **« JK Énergie chargée » / « JK Énergie déchargée »**.

## Installation

### Dans l'add-on ESPHome de Home Assistant
Les packages sont téléchargés depuis ce dépôt public à chaque compilation :
1. Copier **uniquement** `home_energy_management.yaml` dans `/config/esphome/`
   (le renommer `hem-jk.yaml` pour ne pas écraser l'ancien HEM).
2. Ajouter au `secrets.yaml` de HA les clés de [`secrets.yaml.example`](secrets.yaml.example)
   manquantes, notamment `hem_jk_api_encryption_key` et `jk_bms_mac_address`.
3. « Installer » (OTA). Après un push sur `main`, relancer « Installer » suffit.

### En local (CLI)
```bash
cp secrets.yaml.example secrets.yaml   # puis renseigner (clé API UNIQUE)
python -m esphome run home_energy_management.yaml   # version publiée sur GitHub
python -m esphome run local-test.yaml               # fichiers locaux non poussés
```
Sous Windows, compiler depuis PowerShell/cmd (pas Git Bash : ESP-IDF refuse MSys).
`local-test.yaml` duplique les substitutions : répercuter tout changement de pins
dans les deux fichiers.

## Gestion de version

SemVer. Pour publier une version :
1. `project_version` (dans `home_energy_management.yaml`) + `VERSION` + section `CHANGELOG.md`
2. `git commit -m "release: vX.Y.Z"`
3. `git tag -a vX.Y.Z -m "vX.Y.Z"` puis `git push --follow-tags`

La version flashée est visible dans HA (« Version firmware », « Date de compilation »).

## Licence

GPL-2.0 (voir [LICENSE](LICENSE)).
