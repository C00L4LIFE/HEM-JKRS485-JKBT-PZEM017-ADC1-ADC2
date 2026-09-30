# Changelog

Format : [Keep a Changelog](https://keepachangelog.com/fr/1.1.0/) —
versionnage : [SemVer](https://semver.org/lang/fr/) (`MAJEUR.MINEUR.CORRECTIF`).

La version est définie par la substitution `project_version` de
`home_energy_management.yaml` et recopiée dans `VERSION`. Chaque release est
taguée `vX.Y.Z` dans git.

## [Non publié]

## [1.4.0] - 2026-09-30

### Modifié
- **ADC1 = shunt bidirectionnel de l'onduleur hybride** (onduleur + MPPT
  intégré) : `packages/pv2_powermr.yaml` devient `packages/onduleur_adc1.yaml`.
  + = l'onduleur injecte dans le bus DC, − = il tire sur le bus DC.
- « Maison Puissance » = puissance tirée par l'onduleur sur le bus DC.
- « PV2 Seuil production » sert de zone morte dans les deux sens.

### Ajouté
- « Onduleur Courant », « Onduleur Puissance » (signés), « Onduleur Puissance
  injectée / tirée », « Onduleur Énergie injectée / tirée ».
- « Bus DC écart » (diagnostic) : PV + onduleur − JK.

### Conservé
- Capteurs « PV2 … » (mêmes entity_id) : PV2 Puissance = puissance onduleur
  signée, PV2 Puissance (production) / moyenne / Énergie totale = apport net
  du MPPT de l'onduleur.

## [1.3.1] - 2026-09-30

### Ajouté
- PZEM-017 : shunt **150 A** (75 mV). Nouveau sélecteur « PZEM Shunt installé »
  (50/100/150/200/300 A) : le PZEM-017 n'ayant pas de calibre 150 A, il est
  réglé sur 100 A et le courant est multiplié par 1.5 côté ESP (zéro et
  calibration sur référence en tiennent compte).

### Modifié
- L'ancien sélecteur modbus devient « PZEM Calibre (registre PZEM) »
  (diagnostic, piloté automatiquement).

## [1.3.0] - 2026-09-30

### Ajouté
- Heure estimée de fin de charge / décharge (« JK Heure fin estimée »,
  « Batterie Heure fin estimée » pour le JBD) : « Déchargée vers 12:39 »,
  « Chargée vers 15:10 » (date ajoutée si ce n'est pas aujourd'hui).
- Réserve de décharge réglable par batterie (« JK Réserve décharge »,
  « JBD Réserve décharge », défaut 20 %) : en décharge, seule l'énergie
  au-dessus de la réserve est comptée (utile = restante − nominale × réserve).
  La charge n'a aucune compensation.
- JBD : « Batterie Temps restant estimé » rétabli (puissance lissée).

### Modifié
- JK : capacité nominale lue dans la trame réglages
  (`battery_capacity_total_settings`) au lieu d'être déduite du SoC ; l'ancienne
  réserve fixe de 20 Ah est remplacée par la réserve en %.

## [1.2.0] - 2026-09-29

### Ajouté
- **PV2 / PowerMr 60A** (`packages/pv2_powermr.yaml`) : production du 2e
  régulateur via le shunt ADC1 (A0-A1) × tension JK — puissance, production
  filtrée (seuil nuit réglable), moyenne lissée, énergie totale persistante
  (NVS + helper optionnel `input_number.hem_jk_pv2_energy_total`).

### Modifié
- « Maison Puissance » = PV (PZEM-017) + PV2 − puissance JK.

## [1.1.1] - 2026-09-29

### Corrigé
- JK RS485 : les trames cellules (0x02 : tension, courant, SoC, cellules)
  étaient ignorées — `jk_rs485_bms` exige le number `cell_count_settings`
  pour les décoder. Déclaré en `internal` (aucune entité, aucune écriture).
- Câblage JK : A/B inversés côté matériel (corrigé par l'utilisateur).

### Supprimé
- Mode diagnostic (logger DEBUG, dump hexa du bus JK) : logger en INFO,
  sniffer/bms JK en WARN.

## [1.1.0] - 2026-09-29

### Modifié
- Dépôt **public** : `home_energy_management.yaml` charge ses packages depuis
  GitHub (`refresh: 0s`) — dans l'add-on ESPHome de HA, seuls ce fichier et
  `secrets.yaml` sont nécessaires.
- Clé API dédiée `hem_jk_api_encryption_key` (le `secrets.yaml` de HA est
  partagé entre devices).

### Ajouté
- `local-test.yaml` : même config avec packages `!include` locaux, pour tester
  avant de pousser.

## [1.0.1] - 2026-09-29

### Ajouté
- JK BT : mode de connexion configurable — interrupteur persistant
  « JK BT Connexion auto » (ON = connexion automatique au démarrage et
  permanente ; OFF = à la demande avec déconnexion automatique).

### Modifié
- Nouveau JK BMS : MAC Bluetooth A4:C1:38:0A:01:A5 (`secrets.yaml`).

### Corrigé
- PZEM-017 : RX/TX inversés (RX 14 / TX 13) — seule la LED RX du module
  RS485 clignotait, l'ESP émettait sur la broche réception du module.

### Diagnostic (temporaire)
- Logger en DEBUG + dump hexa du bus JK (`uart_jk` debug) : des octets
  arrivent du JK mais aucune trame n'était décodée en v1.0.0.

## [1.0.0] - 2026-09-29

Première version, dérivée de HEM_ESPHOME (`8e44ecc`) et de jk-bms
(`JK-BMS-48V15kWH.yaml`, `b0be6e3`).

### Ajouté
- **JK BMS en RS485, écoute seule** (`packages/jk_bms_rs485.yaml`) sur l'ancien
  port eSmart3 (RX 16, DE/RE 4 forcé bas, pas de TX) via `jk_rs485_sniffer` /
  `jk_rs485_bms` (txubelaxu, commit épinglé). Fournit la référence tension
  batterie du système.
- **JK BMS en Bluetooth pour le paramétrage** (`packages/jk_bms_ble_config.yaml`) :
  interrupteur « JK BT Connexion », déconnecté au démarrage et pendant l'OTA,
  déconnexion automatique réglable.
- **Production solaire PZEM-017** côté panneaux (`packages/pzem017_solar.yaml`) :
  UART2 RX 13 / TX 14 / DE 18, 9600 8N2 ; interface de calibration (calibre
  shunt écrit dans le PZEM, gains/offsets, zéro, calibration sur référence),
  énergie PV persistante (NVS + helper HA `input_number.hem_jk_pv_energy_total`).
- **ADS1115 ADC1 (A0-A1) / ADC2 (A2-A3)** génériques avec calibration
  (`packages/ads1115_adc.yaml`).
- **Gestion de version** : `project_version`, `VERSION`, ce changelog, capteurs
  « Version firmware » et « Date de compilation » (`packages/version.yaml`).

### Modifié
- `energy.yaml` : calculs basés sur le JK (au lieu du JBD / eSmart3) ; maison =
  PV (PZEM-017) − puissance JK.
- `system.yaml` : suppression du watchdog MPPT ; OTA coupe la liaison BLE JK.
- Packages chargés en local (`!include`) : dépôt privé.

### Supprimé
- eSmart3 (composant, package, protection batterie qui pilotait sa consigne
  de courant — plus d'actionneur).
- Shunt MPPT2 et shunt onduleur (rôles retirés des voies ADS1115).
- PZEM-004T (AC) et capteur « DC Load ».
