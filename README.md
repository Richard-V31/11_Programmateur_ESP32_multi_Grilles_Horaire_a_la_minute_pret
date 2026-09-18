<h1 align="center"> Programmateur horaire ESP32 — N relais </h1> 

<h2 align="center">interface web, écran OLED, OTA et Programmation à la minute prêt</h2>

![Platform](https://img.shields.io/badge/Platform-ESP32-green)
![Framework](https://img.shields.io/badge/Framework-Arduino-blue)
![Status](https://img.shields.io/badge/Status-Active-green)
![Release](https://img.shields.io/badge/Release-v1.0.OTA-orange)

Ce programme transforme un **ESP32** en programmateur horaire connecté, capable de piloter un nombre **configurable** de relais indépendants (4 par défaut, extensible sans toucher au code). Chaque relais peut fonctionner en mode **AUTOMATIQUE** (jusqu'à 6 plages horaires librement réglables, à la minute près) ou en mode **MANUEL** (forçage ON/OFF par l'utilisateur, depuis la page web ou un bouton poussoir physique).

Tout est piloté depuis une page web embarquée (servie directement par l'ESP32, sans carte SD) et résumé en temps réel sur un petit écran **OLED SSD1306** — les horaires et les états survivent aux coupures de courant grâce à un enregistrement en mémoire flash (NVS), et le firmware peut être mis à jour par WiFi (**OTA**) sans câble USB.

---

## Sommaire

- [Matériel nécessaire](#matériel-nécessaire)
- [Bibliothèques Arduino nécessaires](#bibliothèques-arduino-nécessaires)
- [Configuration avant le premier flash](#configuration-avant-le-premier-flash)
- [Les plages horaires : une liste libre](#les-plages-horaires--une-liste-libre)
- [L'écran OLED](#lécran-oled)
- [L'interface web](#linterface-web)
- [Sauvegarde des réglages (NVS)](#sauvegarde-des-réglages-nvs)
- [Mise à jour OTA (sans câble USB)](#mise-à-jour-ota-sans-câble-usb)
- [Bouton poussoir physique](#bouton-poussoir-physique)
- [Ajouter ou retirer un relais](#ajouter-ou-retirer-un-relais)
- [Personnalisation rapide](#personnalisation-rapide)

---

## Matériel nécessaire

- Une carte **ESP32** (DevKit classique).
- Un ou plusieurs **modules relais** (typiquement à base de SRD-05VDC, actifs à l'état bas), un par programmation.
- Un écran **OLED SSD1306** 0.96", 128×64 px, en I2C (adresse `0x3C`), câblé sur les broches I2C par défaut de l'ESP32 (SDA = GPIO 21, SCL = GPIO 22).
- Optionnel : un **bouton poussoir** par relais, câblé entre GND et une broche GPIO dédiée (pas de résistance externe nécessaire, la broche est configurée en `INPUT_PULLUP`).

Broches GPIO utilisables en sortie sur un ESP32 DevKit classique : `4, 5, 13, 14, 16, 17, 18, 19, 21, 22, 23, 25, 26, 27, 32, 33` (21/22 étant déjà pris par l'écran OLED). À éviter : GPIO 34-39 (entrée seule) et les broches de boot (0, 2, 12, 15).

## Bibliothèques Arduino nécessaires

À installer depuis le gestionnaire de bibliothèques de l'IDE Arduino :

- `ESPAsyncWebServer` (+ sa dépendance `AsyncTCP`)
- `Preferences` (fournie avec le core ESP32)
- `ArduinoJson`
- `ESPmDNS` (fournie avec le core ESP32)
- `ArduinoOTA` (fournie avec le core ESP32)
- `Adafruit_GFX`
- `Adafruit_SSD1306`

## Configuration avant le premier flash

### 1. Le fichier `arduino_secrets.h`

Ce fichier n'est **pas fourni** (il contient vos identifiants WiFi) : créez-le à côté du `.ino`, avec ce contenu minimal :

```cpp
#define SECRET_SSID  "nom_de_votre_reseau_1"
#define SECRET_PASS  "mot_de_passe_1"
#define SECRET_SSID2 "nom_de_votre_reseau_2"
#define SECRET_PASS2 "mot_de_passe_2"

// Optionnel : jusqu'à un 3e réseau connu
// #define SECRET_SSID3 "nom_de_votre_reseau_3"
// #define SECRET_PASS3 "mot_de_passe_3"

// Optionnel : sécurise les mises à jour OTA (recommandé)
// #define SECRET_OTA_PASSWORD "votre_mot_de_passe_ota"
```

Au démarrage, l'ESP32 scanne les réseaux visibles, ne garde que ceux connus ci-dessus, et se connecte à celui qui offre le meilleur signal (RSSI). Il surveille aussi périodiquement si un réseau habituellement meilleur redevient disponible, pour basculer dessus automatiquement.

### 2. Le tableau `programmateurs[]`

C'est ici que se déclarent les relais. Une ligne = un relais :

```cpp
Programmateur programmateurs[] = {
  { "1", "Programmation 1", "Cuisine",  "#f59e0b", 32, "06:32-08:07,11:30-13:13,18:41-22:30", true, false, 14 },
  { "2", "Programmation 2", "Portail",  "#06b6d4", 33, "07:04-09:08,17:00-19:30",             true, false, 16 },
  { "3", "Programmation 3", "Relais 3", "#0CE892", 25, "",                                    true, false, 17 }, //Pas de programmation
  { "4", "Programmation 4", "Relais 4", "#ef4444", 26, "22:45-06:15",                         true, false, 18 }, //Passage de minuit
};
```

Champs, dans l'ordre : `id` (court, unique, sans espace), `nom` affiché, `sous-titre`, `couleur` (hexadécimale), `broche GPIO`, `plages horaires par défaut` (voir section suivante), `mode auto par défaut`, `état par défaut`, `broche du bouton poussoir` (`-1` = aucun).

Le nombre de relais (`NB_PROGRAMMATEURS`) est déduit automatiquement de la taille de ce tableau : la page web, les routes HTTP, la sauvegarde NVS et l'écran OLED s'adaptent tout seuls.

---

## Les plages horaires : une liste libre

### Principe

Chaque relais, en mode automatique, dispose d'une **liste libre de plages horaires** (jusqu'à `MAX_PLAGES`, 6 par défaut), chacune un simple couple "début → fin" conservé **à la minute près, sans aucun arrondi ni découpage en créneaux** (`06:33-08:17` est parfaitement valide).

Cela permet de définir **autant de plages qu'on le souhaite dans cette limite** pour une même journée (aucune, une, six...), y compris une plage **à cheval sur minuit** (ex: `22:45-06:15`) : ce cas particulier, source classique de bugs dans les programmateurs horaires, est géré nativement.

Le nombre maximum de plages par relais se règle en une seule ligne, en haut du fichier :

```cpp
const int MAX_PLAGES = 6;   // Nombre maximum de plages horaires personnalisées par relais
```

Toute la chaîne — page web, stockage NVS, écran OLED, logique horaire — s'adapte automatiquement à cette valeur, sans autre modification.

### Définir les plages par défaut

Dans le tableau `programmateurs[]`, le champ `plagesDefaut` accepte du texte simple :

| Exemple | Résultat |
|---|---|
| `"09:00-10:30"` | une seule plage |
| `"06:00-08:30,18:00-23:00"` | plusieurs plages, séparées par des virgules |
| `"22:00-06:00"` | une plage à cheval sur minuit |
| `""` | aucune plage (relais éteint en mode auto) |

Les horaires saisis sont conservés tels quels, à la minute près (aucun arrondi). Au-delà de `MAX_PLAGES` plages dans le texte, les suivantes sont ignorées. Ces valeurs par défaut ne servent qu'au tout premier démarrage : dès qu'une programmation a été enregistrée depuis la page web, c'est elle qui prévaut (voir [Sauvegarde des réglages](#sauvegarde-des-réglages-nvs)).

### Modifier les plages depuis la page web

Un appui sur les horaires affichés pour un relais ouvre une fenêtre d'édition : chaque ligne représente une plage, avec deux sélecteurs d'heure (début et fin, réglables à la minute près) et un bouton "✕" pour la supprimer. Un bouton "➕ Ajouter une plage" permet d'en créer de nouvelles jusqu'à `MAX_PLAGES`, et "Tout effacer" vide la liste en un clic. Toutes les plages sont envoyées à l'ESP32 en une seule requête, sous forme d'un texte simple (voir ci-dessous).

### Modifier les plages sans la page web (API)

La route `/save` accepte un texte de plages, pratique en script ou en `curl` :

```bash
curl -X POST "http://richardv.local/save?id=1" -d "plages=06:30-08:00,18:45-22:30"
```

Envoyer `plages=` (valeur vide) efface toutes les plages du relais.

### Résumé lisible des plages

Le firmware sait reconstruire une description texte lisible à partir de la liste de plages (utilisée à la fois par la page web et par l'écran OLED) : `"06:30-08:00, 11:30-13:15 +1"` par exemple, où `+1` indique qu'une plage supplémentaire existe mais n'est pas détaillée. Ce même mécanisme calcule aussi, à tout moment, le nombre de minutes restant avant le **prochain changement d'état** du relais (affiché sur la page web sous la forme "Extinction dans..."/"Allumage dans...").

---

## L'écran OLED

L'écran (128×64 px, ~21 caractères par ligne en taille de police par défaut) affiche en continu :

- **Ligne 1** : le nom du réseau WiFi utilisé, et l'heure courante (`HH:MM`).
- **Ligne 2** : l'adresse IP locale de l'ESP32.
- **Ligne 3** : l'indicateur de page (`Page 1/2`, etc.) si l'affichage doit tourner sur plusieurs pages — voir plus bas —, sinon une ligne vide.
- **Une ligne par relais** (jusqu'à `OLED_LIGNES_PAR_PAGE`, 5 par défaut).

### Format d'une ligne relais

Chaque ligne relais tient sur les 21 caractères de large, et prend la forme :

```
1 A 08:00>11:30-13:15
```

- **L'identifiant du relais** (`1`) est affiché en **vidéo inverse** (fond blanc, texte noir) quand le relais est **actif (ON)**, et en texte normal quand il est **OFF** — il n'y a donc plus besoin d'écrire le mot "ON"/"OFF" en toutes lettres, ce qui libère de la place pour les horaires.
- **La lettre de mode** : `A` pour Automatique, `M` pour Manuel.
- **Les horaires**, uniquement en mode automatique (rien n'est affiché ici en mode manuel, puisqu'aucun horaire ne s'applique) :
  - si le relais est **actuellement actif** : `HEURE_FIN>DEBUT-FIN` — l'heure à laquelle il va s'éteindre, puis la plage suivante en entier. Exemple : `08:00>11:30-13:15` signifie "s'éteint à 08:00, puis la prochaine plage va de 11:30 à 13:15".
  - si le relais n'est **pas encore actif** : `DEBUT-FIN` — la prochaine plage à venir, affichée en entier. Exemple : `11:30-13:15` signifie "prochaine plage de 11:30 à 13:15".
  - cas particuliers : `24h/24` (plages couvrant toute la journée sans interruption) et `aucune plage` (aucune plage définie).

On ne réaffiche pas le début de la plage en cours (déjà passé, donc peu utile) ni la fin de la plage d'après-la-suivante : c'est le compromis qui permet de tout faire tenir sur une seule ligne de 21 caractères, tout en gardant l'identifiant du relais, son mode et son état.

Si l'heure n'est pas encore synchronisée par NTP au moment de l'affichage, l'écran retombe sur un résumé de secours : la première plage définie pour ce relais, suivie de `+n` s'il y en a d'autres (ex: `06:30-08:00+2`).

### Pagination automatique

Si le nombre de relais déclarés dépasse `OLED_LIGNES_PAR_PAGE` (5 par défaut), l'affichage bascule automatiquement en **pages tournantes** : chaque page reste affichée 8 secondes avant de passer à la suivante, en boucle. Avec 4 relais (configuration par défaut), tout tient sur une seule page et l'indicateur de page ne s'affiche pas.

### Écrans spéciaux

L'écran OLED affiche aussi, ponctuellement : "Démarrage..." et "Synchro heure..." pendant l'initialisation, "Pas de réseau WiFi connu. Redémarrage..." en cas d'échec de connexion, puis une barre de progression et un message de confirmation pendant et après une mise à jour OTA.

---

## L'interface web

La page (accessible via `http://<hostname>.local` ou l'adresse IP de l'ESP32) est entièrement embarquée dans le firmware (HTML/CSS/JS en mémoire flash) et s'adapte automatiquement au nombre de relais déclarés, via ces routes HTTP :

| Route | Méthode | Rôle |
|---|---|---|
| `/` | GET | Sert la page principale. |
| `/get-config` | GET | Décrit les relais à afficher (id, nom, couleur...) et le nombre maximum de plages par relais (`MAX_PLAGES`). |
| `/get-data` | GET | État complet de tous les relais (plages, résumé, mode, état, minutes avant le prochain changement), interrogée chaque seconde. |
| `/get-info` | GET | Informations système (WiFi, IP, MAC, signal, date de build) pour le popup "Infos système". |
| `/toggle-mode?id=...` | GET | Bascule un relais entre Automatique et Manuel. |
| `/force-state?id=...` | GET | Force l'état ON/OFF d'un relais (et passe en mode Manuel). |
| `/save?id=...` | POST | Enregistre les nouvelles plages d'un relais (`plages=<texte>`). |
| `/reset-auto` | GET | Repasse tous les relais en mode Automatique en une fois. |

Chaque bloc relais de la page affiche son nom, son état (voyant ON/OFF), le bouton de bascule Auto/Manuel, le résumé de ses plages (avec la plage en cours en vert et la suivante en rouge), et le temps restant avant le prochain changement.

---

## Sauvegarde des réglages (NVS)

Les plages horaires, modes et états sont enregistrés dans la mémoire flash NVS (bibliothèque `Preferences`), namespace `config`, avec une clé par relais et par type d'information (ex: `1.pl` pour les plages, `1.auto` pour le mode, `1.etM` pour l'état) — ils survivent donc aux coupures de courant et aux redémarrages.

**Point d'attention** : à chaque mise à jour OTA, le firmware compare sa date de compilation (`FIRMWARE_BUILD`) à celle enregistrée en NVS lors du démarrage précédent. Si elle a changé, la NVS est automatiquement réinitialisée, pour que les nouvelles valeurs par défaut du tableau `programmateurs[]` prennent bien effet (sans cela, d'anciens réglages enregistrés continueraient de masquer silencieusement toute modification faite dans le code).

---

## Mise à jour OTA (sans câble USB)

Une fois l'ESP32 flashé une première fois par USB et connecté au WiFi, les mises à jour suivantes peuvent se faire directement depuis l'IDE Arduino (menu **Outils > Port**), tant que l'ordinateur et l'ESP32 sont sur le même réseau. Pensez à définir `SECRET_OTA_PASSWORD` dans `arduino_secrets.h` pour sécuriser cet accès (sinon un message d'avertissement s'affiche au démarrage, dans le moniteur série).

La date/heure de compilation (`FIRMWARE_BUILD`) s'affiche au démarrage (moniteur série et écran OLED) et dans le popup "Infos système" de la page web : c'est le seul moyen fiable de vérifier qu'une mise à jour a bien pris effet.

---

## Bouton poussoir physique

Si une broche `pinBP` est déclarée pour un relais (câblage : une patte du bouton sur GND, l'autre sur la broche GPIO indiquée), chaque appui bref inverse l'état du relais **et** bascule automatiquement en mode manuel — exactement comme le bouton "Forcer ON/OFF" de la page web. Un anti-rebond logiciel (40 ms par défaut) évite les faux déclenchements.

---

## Ajouter ou retirer un relais

Il suffit de dupliquer (ou supprimer) une ligne du tableau `programmateurs[]` : id unique, broche GPIO libre, et éventuellement une broche de bouton poussoir. Aucune autre partie du code n'a besoin d'être modifiée — page web, routes HTTP, sauvegarde NVS et écran OLED (pagination comprise) s'adaptent automatiquement au nouveau nombre de relais.

## Personnalisation rapide

| Constante | Rôle | Défaut |
|---|---|---|
| `MAX_PLAGES` | Nombre maximum de plages horaires personnalisées par relais | `6` |
| `OLED_LIGNES_PAR_PAGE` | Nombre de lignes "relais" affichées par page sur l'écran OLED | `5` |
| `hostname` | Nom d'accès local (`http://<hostname>.local`) et nom OTA | `"richardv"` |
| `TZ_INFO` | Fuseau horaire (gère automatiquement l'heure d'été/hiver) | France (`CET-1CEST,M3.5.0,M10.5.0/3`) |
