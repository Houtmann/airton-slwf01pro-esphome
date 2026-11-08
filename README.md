# Piloter une climatisation Airton dans Home Assistant

**Sans cloud, sans abonnement, sans soudure** — avec un module Wi-Fi du commerce reflashé en ESPHome.

Ce dépôt documente l'intégration d'un climatiseur **Airton** utilisant le protocole **TuyaMCU**, à l'aide d'un dongle **SMLIGHT SLWF-01 Pro** reflashé. Le module se branche sur le port USB de l'unité intérieure : aucun câblage, aucun composant à souder.

Il contient la configuration finale, deux firmwares de diagnostic, la table complète des datapoints, et surtout **les six pièges** sur lesquels on perd des heures.

---

## Sommaire

- [Est-ce que ça concerne ma clim ?](#est-ce-que-ça-concerne-ma-clim-)
- [Matériel](#matériel)
- [Pourquoi ce n'est pas immédiat](#pourquoi-ce-nest-pas-immédiat)
- [Installation pas à pas](#installation-pas-à-pas)
- [Table des datapoints](#table-des-datapoints)
- [Les six pièges](#les-six-pièges)
- [Ce qui ne fonctionne pas](#ce-qui-ne-fonctionne-pas)
- [Crédits](#crédits)

---

## Est-ce que ça concerne ma clim ?

Les Airton se répartissent en **deux familles incompatibles entre elles**, et rien sur l'emballage ne le dit :

| Famille | Protocole | Solution |
|---|---|---|
| Modules **409945** et assimilés | **TuyaMCU** (`55 AA ...`) | **ce dépôt** |
| Autres modèles Airton | protocole propriétaire (`7A 7A ...`) | [devildant/acw02_esphome](https://github.com/devildant/acw02_esphome) |

**Comment trancher sans rien démonter :** regardez avec quelle application votre clim était censée fonctionner.

- **Smart Life / Tuya** → TuyaMCU, vous êtes au bon endroit
- **NetHome Plus / Midea Air** → protocole Midea, ce dépôt ne s'applique pas

En cas de doute, le firmware de diagnostic de l'[étape 1](#étape-1--identifier-le-protocole) répond en trente secondes, sans rien risquer.

> Ces cartes sont fabriquées par **Midea** et rebadgées. La même méthode vaut donc pour d'autres marques du groupe : Teknopoint, et probablement Qlima, Comfee ou Artel.

---

## Matériel

**Un seul composant : le [SMLIGHT SLWF-01 Pro](https://smlight.tech/product/slwf-01), version 2.1**, une dizaine d'euros — [chez Domadoo](https://www.domadoo.fr/fr/controle-chauffage-clim/7774-smlight-module-wi-fi-pour-climatiseur-slwf-01-pro.html) par exemple.

C'est un dongle USB qui remplace le module Wi-Fi d'origine sur l'unité intérieure. La v2.1 se reconnaît à son **port USB-C** supplémentaire, qui permet de la flasher sans démontage compliqué.

| Caractéristique | Valeur |
|---|---|
| Microcontrôleur | ESP8266 (`board: esp12e`) |
| UART | GPIO12 (TX) / GPIO14 (RX), 9600 bauds |
| Alimentation | fournie par le port USB de la clim |

> **Alimentation** : selon des retours d'autres utilisateurs (non vérifié ici), certaines cartes Airton fournissent trop peu de courant sur le connecteur, ce qui provoque des reboots ou un Wi-Fi instable. Contournement rapporté : alimenter le module en 5 V externe, avec une masse commune. Voir le README de [devildant/acw02_esphome](https://github.com/devildant/acw02_esphome).

> **Attention** : le SLWF-01 Pro est vendu avec un firmware **Midea** préinstallé. Sur une clim Tuya, il ne fonctionnera pas — voir ci-dessous.

---

## Pourquoi ce n'est pas immédiat

Le module sort d'usine avec un firmware ESPHome utilisant le composant `midea`. Branché sur une carte Tuya, il émet des trames parfaitement formées… que la carte ne comprend pas. Elle ne répond jamais.

Le résultat est déroutant : le module apparaît en ligne dans Home Assistant, son signal Wi-Fi et son temps de fonctionnement se rafraîchissent normalement, **mais toutes les entités de la climatisation restent à `unknown`**. Les logs ne montrent que des `Response timeout` en boucle.

Un indice permet de reconnaître ce cas immédiatement : `fan_mode` affiche `on`, une valeur qui n'existe même pas dans la liste des modes valides. C'est la valeur par défaut d'ESPHome quand **aucune trame n'a jamais été reçue**.

Ce n'est ni un défaut de branchement ni un module défectueux : **c'est le mauvais protocole**. Reflasher règle le problème.

---

## Installation pas à pas

### Prérequis

L'add-on **ESPHome Device Builder** dans Home Assistant (Paramètres → Modules complémentaires → dépôt ESPHome), en version **2026.10.1 ou plus récente** recommandée (voir [piège n°5](#5-version-desphome-et-uart-logicielle)).

Copiez `secrets.yaml.example` en `secrets.yaml` et renseignez vos identifiants Wi-Fi. Ce fichier est ignoré par git.

### Étape 1 — Identifier le protocole

Flashez [`diagnostic/01-sniffer-uart.yaml`](diagnostic/01-sniffer-uart.yaml). Il n'agit sur rien, il écoute.

Ouvrez les logs et observez **trente secondes sans rien toucher** :

| Ce que vous voyez | Verdict |
|---|---|
| `55 AA 03 07 ...` | **TuyaMCU** — continuez à l'étape 2 |
| `AA xx AC ...` | Midea — le firmware d'origine convenait |
| rien du tout | bus muet, voir plus bas |

Une carte Tuya **émet spontanément**, plusieurs fois par seconde. Une carte Midea reste silencieuse tant qu'on ne l'interroge pas : d'où les deux boutons de sonde inclus dans le firmware.

**Si rien n'arrive**, le problème est physique. Dans l'ordre : coupez le disjoncteur trente secondes module branché (la carte n'ouvre son UART qu'au démarrage) ; réenfoncez la clé à fond (l'alimentation passe par les broches d'extrémité, les données par celles du milieu — un module à moitié enfoncé s'allume sans transmettre) ; vérifiez qu'il n'existe pas un second port USB de service, dont les broches ne seraient pas câblées.

### Étape 2 — Énumérer les datapoints

Flashez [`diagnostic/02-tuya-discovery.yaml`](diagnostic/02-tuya-discovery.yaml). La poignée de main Tuya s'effectue et la liste complète s'affiche :

```
[tuya] Datapoint 1: switch (value: ON)
[tuya] Datapoint 2: int value (value: 240)
[tuya] Datapoint 4: enum (value: 1)
[tuya] Product: '{"p":"pjnywabwywopufvb","v":"1.0.0","m":0}'
```

Manipulez ensuite la télécommande **une fonction à la fois** en regardant quel datapoint bouge. C'est ainsi qu'on cartographie une carte inconnue.

Si vos valeurs correspondent à la [table ci-dessous](#table-des-datapoints), passez directement à l'étape 4.

### Étape 3 — Cartographier ce qui reste (optionnel)

Si votre climatiseur n'expose pas exactement les mêmes datapoints, ou pour identifier ceux qui restent inconnus, flashez [`diagnostic/03-cartographie-datapoints.yaml`](diagnostic/03-cartographie-datapoints.yaml).

Cette configuration donne de quoi piloter la clim pendant les essais **et** expose tous les datapoints non identifiés en lecture seule. Ouvrez la page de l'appareil dans Home Assistant, appuyez sur un bouton à la fois, et regardez lequel bouge.

> La lecture seule est volontaire : écrire à l'aveugle sur un datapoint inconnu d'une carte de climatisation peut déclencher n'importe quoi, y compris des modes de service.

### Étape 4 — La configuration finale

Flashez [`airton-clim.yaml`](airton-clim.yaml). Vous obtenez dans Home Assistant : un thermostat, cinq modes, huit vitesses de ventilation, les deux oscillations, le mode nuit, le purificateur, l'affichage, la température de la pièce et les codes défaut.

La configuration qui tourne réellement chez moi est [`airton-clim-complet.yaml`](airton-clim-complet.yaml) (étape 9.9) : même base, plus une resynchronisation au boot qui attend la température, une resync périodique, la correction DP3 (+256), et des capteurs de diagnostic (boîte noire avant reboot, compteurs de trames aberrantes). Son en-tête retrace l'historique des étapes. Le nom du device y reste `slwf-sniffer`, pour conserver les identifiants d'entités.

> ⚠️ **Éprouvé partiellement.** Les datapoints 1, 2, 3, 4, 5 et 20 ont été vérifiés directement sur le matériel. Les datapoints **13, 105, 106, 107, 109 et 110** proviennent de la [documentation de devildant/acw02_esphome](https://github.com/devildant/acw02_esphome/blob/main/docs/TUYA_MCU_SETUP.md) et n'ont pas encore été retestés ici, pas plus que le bloc `active_state`. En cas de comportement inattendu, repliez-vous sur la configuration de l'étape 3, qui ne contient que du vérifié.

### Flasher, concrètement

Le firmware d'usine est protégé par un mot de passe OTA inconnu : **le premier flash passe obligatoirement par l'USB.**

1. Dans ESPHome : `Install` → `Manual download`, récupérez le `.bin`
2. Débranchez la clé de la clim, branchez-la en **USB-C sur votre ordinateur, bouton maintenu enfoncé**
3. Sous Chrome, allez sur [web.esphome.io](https://web.esphome.io) → `Connect` → `Install`
4. Rebranchez dans la clim

Si le port n'apparaît pas, installez le [pilote CH340](https://learn.sparkfun.com/tutorials/how-to-install-ch340-drivers/all).

> Un avertissement `Flash ID: 0` / `Failed to communicate with the flash chip` est **sans conséquence** : l'effacement et l'écriture se déroulent normalement.

**Les flashs suivants se font en OTA**, sans démontage : les configurations de ce dépôt ne définissent pas de mot de passe OTA.

---

## Table des datapoints

| DP | Type | Fonction | Remarque |
|---|---|---|---|
| 1 | switch | Marche / arrêt | |
| 2 | int | Consigne | **en dixièmes** (240 = 24,0 °C) |
| 3 | int | Température ambiante | **en dixièmes**, voir piège n°3 |
| 4 | enum | Mode | `0` auto · `1` froid · `2` déshu · **`3` chaud** · **`4` ventilation** |
| 5 | enum | Ventilation | `0` auto · `1`→`5` vitesses · `6` silencieux · `7` turbo |
| 13 | switch | Affichage | **write-only**, jamais rapporté — voir piège n°4 |
| 20 | bitmask | Codes défaut | `0` = aucun défaut |
| 105 | enum | Unité | `0` °C · `1` °F |
| 106 | enum | Oscillation horizontale | `0` off · `1` on |
| 107 | enum | Oscillation verticale | `0` off · `1` on · `2`→`6` positions |
| 109 | switch | Mode nuit | |
| 110 | switch | Purificateur (HEALTH) | |

**Datapoints rapportés mais non identifiés** : `8`, `12`, `21`, `22`, `101`→`104`, `108`, `111`→`115`.
Les boutons **ECO/SC**, **I FEEL** et **CLEAN** de la télécommande s'y trouvent probablement. Les contributions sont bienvenues.

> ⚠️ **L'ordre des modes est inhabituel** : `3` = chaud et `4` = ventilation, soit l'inverse du mapping Tuya le plus répandu. Vérifié sur le matériel. Si votre clim ventile quand vous demandez du chauffage, permutez ces deux valeurs.

---

## Les six pièges

### 1. `rx_buffer_size` — le plus coûteux en temps

GPIO12 et GPIO14 ne sont pas les broches de l'UART matériel de l'ESP8266 : ESPHome bascule en **software serial**, dont le tampon par défaut ne fait que **256 octets**.

La carte émettant des dizaines de trames par seconde, ce tampon déborde. On perd alors des octets **au milieu d'une trame**, le décodeur se désynchronise et produit des valeurs fantômes parfaitement plausibles :

```
Consigne réelle 270  ->  encodage 00 00 01 0E
Octet 01 perdu       ->  lu comme 0E = 14
```

Symptômes : températures absurdes, consigne affichée sans rapport avec la clim, trames tronquées du type `55 A0 10 08` au lieu de `55 AA 03 07`.

```yaml
uart:
  rx_buffer_size: 1024   # au lieu de 256
```

Un piège aggravant : passer le composant `tuya` en `DEBUG` fait journaliser chaque mise à jour de datapoint, des dizaines de fois par seconde. Produire ces logs et les pousser par Wi-Fi consomme assez de temps processeur pour **faire déborder le tampon** — le remède devient la cause. Restez en `INFO` en usage normal.

> Sur ESP32 avec UART matériel, ce piège n'existe pas.

### 2. `temperature_multiplier`

Consigne et mesure sont exprimées en dixièmes de degré. Sans multiplicateur, votre thermostat affiche **240 °C**.

Et ESPHome refuse `target_temperature_multiplier` employé seul :

```
current_temperature_multiplier required if using target_temperature_multiplier
```

Utilisez la clé unique, qui couvre les deux :

```yaml
temperature_multiplier: 0.1
```

### 3. « I FEEL » et la double sonde

Si votre télécommande possède un bouton **I FEEL**, elle mesure la température là où elle se trouve et l'envoie à la clim. Celle-ci publie alors **deux valeurs différentes sur le même DP3**, en alternance, plusieurs fois par seconde :

```
55 AA 03 07 00 08 03 02 00 04 00 00 00 E6 00   -> 230 = 23,0 °C
55 AA 03 07 00 08 03 02 00 04 00 00 00 F0 0A   -> 240 = 24,0 °C
```

Les deux trames sont valides, sommes de contrôle comprises. **Ce n'est pas une corruption** : la carte émet réellement les deux, sans le moindre discriminant permettant de les distinguer.

Branché tel quel sur `current_temperature_datapoint`, le résultat est doublement pénible. Le thermostat compare la mesure à la consigne et **bascule entre `HEATING` et `IDLE` des dizaines de fois par seconde** ; et chaque bascule étant un changement d'état, la base `recorder` de Home Assistant enfle de dizaines de milliers de lignes par heure.

La parade retenue ici : sortir la température du thermostat et l'exposer via un capteur filtré.

```yaml
filters:
  - multiply: 0.1
  - median: {window_size: 15, send_every: 15}
  - throttle_average: 30s
```

On obtient une valeur stable — la moyenne des deux sondes — publiée toutes les trente secondes.

> Désactiver I FEEL sur la télécommande devrait supprimer le problème à la source. Le datapoint correspondant n'a pas encore été identifié.

### 4. Les datapoints write-only

**DP13 (Affichage) n'apparaît dans aucune énumération.** Ni au démarrage, ni dans les rapports périodiques, ni après une action sur la télécommande. Il est déclaré write-only : la carte exécute la commande sans jamais renvoyer d'état.

Impossible à découvrir par observation — il faut le savoir. ESPHome conservera la dernière valeur commandée, qui peut différer de la réalité si vous utilisez la télécommande.

Corollaire : **l'énumération initiale n'est pas exhaustive**. Le DP12, absent au démarrage, est apparu au premier changement de mode. D'autres datapoints dorment probablement encore.

### 5. Version d'ESPHome et UART logicielle

Comme vu au piège n°1, GPIO12/14 imposent une **UART logicielle**. La [documentation UART d'ESPHome](https://esphome.io/components/uart/) ne liste comme broches matérielles sur ESP8266 que GPIO1/3 et GPIO15/13.

Avant ESPHome 2026.10, la réception logicielle bloquait environ **1 ms par octet avec les interruptions coupées** ([PR esphome#18519](https://github.com/esphome/esphome/pull/18519)). Pendant ce temps, le Wi-Fi et le watchdog sont privés de temps processeur.

Symptômes : octets perdus, erreurs de checksum dans les logs (`55!=xx`), watchdogs, redémarrages inexpliqués.

La 2026.10 décode la réception par fronts. **ESPHome 2026.10.1 ou plus récent est donc recommandé** : cela corrige un défaut connu, sans prétendre éliminer toute perte d'octets (voir « Ce qui ne fonctionne pas »). Le correctif est en test sur le matériel de l'auteur depuis le 30/10/2026 ; il n'y a pas encore de recul.

En complément, la configuration fixe `wifi: power_save_mode: none` : le mode `LIGHT` par défaut de l'ESP8266 aggrave les pertes d'octets.

### 6. La température qui retombe à 0 au-delà de 25,5 °C

Le MCU Airton encode la température ambiante (DP3) en **uint8 de dixièmes de degré** : la valeur maximale est 255, soit 25,5 °C. Au-delà, elle déborde et repart de zéro : **26,0 °C arrive sous la forme `4`** (260 − 256), soit 0,4 °C une fois le multiplicateur appliqué.

Le correctif est un filtre placé avant le `multiply` : une valeur inférieure à 70 (7,0 °C) est impossible en intérieur, on lui ajoute donc 256.

```yaml
filters:
  - lambda: |-
      if (x < 70) x += 256;
      return x;
  - multiply: 0.1
```

Source : le fil HACF [« Module WIFI climatisation Airton »](https://forum.hacf.fr/t/module-wifi-climatisation-airton/20958).

---

## Ce qui ne fonctionne pas

**Aucune mesure de consommation.** La carte n'expose aucun datapoint de puissance ou d'énergie, et aucun projet connu n'en a trouvé. Pour suivre la consommation de votre clim, il faut un compteur externe.

**Trois boutons non cartographiés** : ECO/SC, I FEEL et CLEAN. Ils existent parmi les datapoints listés plus haut, mais leur correspondance reste à établir.

**Le mode HVAC affiché reste imparfait.** Le composant `climate` de Tuya n'a pas été conçu pour des climatiseurs multi-modes ; utilisez le sélecteur **Mode** plutôt que la carte thermostat pour changer de mode.

**Quelques trames restent occasionnellement décalées** malgré le tampon agrandi (`55 55 AA 03 07 ...`). Le composant rejette la trame et lit la suivante : sans conséquence à ce débit.

---

## Crédits

- [**devildant/acw02_esphome**](https://github.com/devildant/acw02_esphome) — le projet de référence pour les Airton, dont la [documentation TuyaMCU](https://github.com/devildant/acw02_esphome/blob/main/docs/TUYA_MCU_SETUP.md) a permis de confirmer le mapping et d'identifier les DP 13, 105, 106, 107, 109 et 110. Si votre clim utilise le protocole `7A 7A`, c'est là qu'il faut aller.
- [**@mrgnsrl**](https://github.com/mrgnsrl) — identification initiale des datapoints TuyaMCU sur Airton.
- [**SMLIGHT**](https://github.com/smlight-tech/slwf-01pro-esphome) — configuration matérielle du SLWF-01 Pro (brochage UART, procédure de flash).
- [**Composant Tuya d'ESPHome**](https://esphome.io/components/tuya/).

Ce que ce dépôt apporte en propre : l'usage d'un **dongle du commerce sans soudure** en lieu et place d'un montage ESP32 sur PCB, le réglage `rx_buffer_size` propre à l'ESP8266 en software serial, et le diagnostic de la double sonde « I FEEL ».

## Licence

MIT — voir [LICENSE](LICENSE).
