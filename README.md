#  POZOS — Student List

> POC Docker pour l'application `student_list` de POZOS — déploiement découplé, scalable et automatisé via Docker et Docker Compose, avec mise en place d'un registry privé.

---

## 📋 Table des matières

- [Contexte](#-contexte)
- [Architecture](#-architecture)
- [Prérequis](#-prérequis)
- [Structure du projet](#-structure-du-projet)
- [Déploiement](#-déploiement)
  - [1. Application (API + Website)](#1-application-api--website)
  - [2. Registry privé](#2-registry-privé)
  - [3. Push de l'image sur le registry](#3-push-de-limage-sur-le-registry)
- [Choix techniques](#-choix-techniques)
- [Problèmes rencontrés](#-problèmes-rencontrés)
- [Nettoyage](#-nettoyage)

---

##  Contexte

POZOS est une entreprise IT française qui développe des logiciels pour les lycées. L'application `student_list` permet d'afficher la liste des élèves avec leur âge.

L'objectif de ce POC est de **dockeriser** l'application existante pour la rendre :
- **Scalable** — chaque module est indépendant et peut être redéployé ou répliqué isolément
- **Facilement déployable** — un seul `docker compose up -d` suffit à lancer toute la stack
- **Versionnable** — les images sont stockées dans un registry privé pour assurer la traçabilité des versions

L'application est composée de **deux modules découplés** :
- Une **API REST** en Python/Flask avec authentification basique
- Un **frontend web** en PHP qui consomme l'API

Une **infrastructure de stockage d'images** (registry privé) complète le dispositif pour répondre aux besoins de versionnement des releases.

---

##  Architecture

```
┌──────────────────────────────────────────────┐
│           Stack applicative                  │
│        Réseau: pozos_network                 │
│                                              │
│   ┌──────────────┐         ┌──────────────┐  │
│   │   website    │ ──────> │     api      │  │
│   │  (php:apache)│         │ (Flask 3.x)  │  │
│   │   port 80    │         │   port 5000  │  │
│   └──────────────┘         └──────────────┘  │
│         │                         │          │
└─────────┼─────────────────────────┼──────────┘
          │                         │
       8080:80                  5000:5000
          │                         │
   ┌──────┴────┐            ┌───────┴───┐
   │  Browser  │            │   curl    │
   └───────────┘            └───────────┘


┌──────────────────────────────────────────────┐
│        Stack registry (séparé)               │
│      Réseau: registry_network                │
│                                              │
│   ┌──────────────┐         ┌──────────────┐  │
│   │  registry-ui │ ──────> │   registry   │  │
│   │   (Joxit)    │         │  (registry:2)│  │
│   │   port 80    │         │   port 5000  │  │
│   └──────────────┘         └──────────────┘  │
└──────────────────────────────────────────────┘
       8085:80                   5050:5000
```

Les deux stacks sont **volontairement séparés** : le registry relève de l'infrastructure et a un cycle de vie indépendant de l'application.

---

## Prérequis

- **Docker Engine** ≥ 20.x
- **Docker Compose** v2 (commande `docker compose` avec espace, le plugin intégré au CLI Docker)
- **Git**
- Ports libres sur la machine hôte : `5000`, `8080`, `5050`, `8085`

Vérification rapide :

```bash
docker --version
docker compose version
```

---

## Structure du projet

```
student-list/
├── README.md                       ← ce fichier
├── docker-compose.yml              ← stack applicative (API + Website)
├── simple_api/
│   ├── Dockerfile                  ← build de l'image API
│   ├── student_age.py              ← code Flask
│   ├── student_age.json            ← données (montées en volume)
│   └── requirements.txt            ← dépendances Python
├── website/
│   └── index.php                   ← frontend (URL de l'API à configurer)
├── registry/
│   └── docker-compose.yml          ← stack registry privé
└── screenshots/                    ← captures d'écran du livrable
```

---

## Déploiement

### Cloner le repo

```bash
git clone https://github.com/Binfl/student-list.git
cd student-list
```

---

### 1. Application (API + Website)

#### Build de l'image API et lancement du stack

Depuis la racine du projet :

```bash
docker compose up -d
```

Au premier lancement, Docker va builder l'image `student-list-api:v1` à partir du `Dockerfile` situé dans `simple_api/`. Cela prend 1 à 2 minutes (téléchargement de `python:3.13-slim`, installation des paquets système et Python).

Vérifier que les deux services sont `Up` :

```bash
docker compose ps
```

**Capture : les deux conteneurs `student-list_api` et `student-list_web` actifs**

![test1](https://media.discordapp.net/attachments/878026931025608734/1498010880451809300/image.png?ex=69ef9ad9&is=69ee4959&hm=75c4a5d2053f9dd0dead20eae7d520b9a9a251fbd63c036d832bcb6c78cf589a&=&format=webp&quality=lossless)

#### Test de l'API en isolation

```bash
curl -u toto:python -X GET http://localhost:5000/pozos/api/v1.0/get_student_ages
```

Réponse attendue :

```json
{
  "student_ages": {
    "alice": "12",
    "bob": "13"
  }
}
```

Le `-u toto:python` envoie l'authentification basique configurée dans `student_age.py`.

📸 **Capture : test curl de l'API renvoyant le JSON**

![test4](https://media.discordapp.net/attachments/878026931025608734/1498014141594734702/image.png?ex=69ef9de3&is=69ee4c63&hm=f6c677213b5797452a476f71ce9e52600a8b67ecf3025f999eb65407d723433e&=&format=webp&quality=lossless)

#### Test du frontend

Ouvrir un navigateur sur :

```
http://localhost:8080
```

Cliquer sur le bouton **"List Student"** — la liste des élèves doit apparaître.

📸 **Capture : page POZOS avec la liste affichée (alice 12, bob 13)**

![test2](https://media.discordapp.net/attachments/878026931025608734/1497998356125974578/scree_docker_livrable_2.png?ex=69ef8f2f&is=69ee3daf&hm=ab076e385d880cc08d61e7a3101b3d9d4fc38de8d2a7064f405b97f5d3dc34d2&=&format=webp&quality=lossless)



À ce stade, l'application complète tourne en conteneurs avec :
- Communication inter-services via le réseau Docker `pozos_network`
- Données externalisées (`student_age.json`) montées en volume
- Frontend découplé du backend

---

### 2. Registry privé

Le registry est dans un **stack séparé**. Voir [Choix techniques](#-choix-techniques) pour la justification.

```bash
cd registry
docker compose up -d
docker compose ps
```

📸 **Capture : `registry` et `registry-ui` actifs**

![test5](https://media.discordapp.net/attachments/878026931025608734/1498010743302521064/image.png?ex=69ef9ab8&is=69ee4938&hm=262cc5dec7299963b6bb197ccce20cd6ee915b76f6b2b2ae0940fd9e578d6645&=&format=webp&quality=lossless)

L'UI web est accessible sur :

```
http://localhost:8085
```

Au premier démarrage, l'UI affichera "0 repository" car aucune image n'a encore été poussée.

---

### 3. Push de l'image sur le registry

Depuis la racine du projet (revenir d'abord avec `cd ..`) :

#### Tagger l'image pour le registry local

```bash
docker tag student-list-api:v1 localhost:5050/student-list-api:v1
```

Vérification :

```bash
docker images | grep student-list-api
```

Les deux entrées doivent partager le même `IMAGE ID` (c'est la même image avec deux noms).

#### Pousser l'image

```bash
docker push localhost:5050/student-list-api:v1
```

Sortie attendue :

```
The push refers to repository [localhost:5050/student-list-api]
abc123: Pushed
def456: Pushed
v1: digest: sha256:... size: ...
```

Rafraîchir `http://localhost:8085` — l'image `student-list-api:v1` apparaît dans la liste, avec sa taille, son architecture (`amd64`), et la date du push.

📸 **Capture : UI Joxit affichant l'image `student-list-api:v1`**

![test6](https://media.discordapp.net/attachments/878026931025608734/1498007445354578110/image.png?ex=69ef97a6&is=69ee4626&hm=f082121bbe309a407a34681848ad3035842cdd5668a3128b46eb8e2623137de6&=&format=webp&quality=lossless)

#### Vérification via l'API du registry

Optionnel, mais utile pour valider le bon fonctionnement :

```bash
# Lister les repositories
curl http://localhost:5050/v2/_catalog

# Lister les tags d'un repository
curl http://localhost:5050/v2/student-list-api/tags/list
```

Sortie attendue :

```json
{"repositories":["student-list-api"]}
{"name":"student-list-api","tags":["v1"]}
```

📸 **Capture (optionnelle) : sortie des appels curl à l'API du registry**

> _Insérer ici la capture (facultative)._

---

## 🛠️ Choix techniques

### Image de base `python:3.13-slim` pour l'API

Imposée par l'énoncé. La variante "slim" réduit significativement la taille de l'image finale (~150 Mo au lieu de ~900 Mo pour la full), au prix d'un environnement plus minimaliste qui nécessite d'installer manuellement quelques outils (voir le point sur `build-essential`).

### Installation de `build-essential` dans le Dockerfile

Le paquet Python `python-ldap` (dépendance transitive de `flask-simpleldap`) contient du code C qui doit être compilé à l'installation. L'image `python:3.13-slim` ne fournit pas de compilateur, d'où l'ajout de `build-essential` (qui inclut `gcc`, `make`, et les headers C standards) dans la liste apt-get.

### `php:apache` plutôt que `httpd:2.4` pour le website

L'image `httpd:2.4` n'inclut **pas le module PHP** — un fichier `.php` serait servi en texte brut au navigateur. `php:apache` intègre PHP et Apache déjà configurés ensemble, prêts à l'emploi pour des sites PHP simples.

### Bind mount sur `./website:/var/www/html`

Plutôt que de builder une image custom pour le frontend, on injecte simplement le code PHP dans l'image officielle via un bind mount. Avantages :
- Pas de Dockerfile à maintenir pour le frontend
- Modifications de `index.php` prises en compte sans rebuild
- Image officielle régulièrement maintenue par les équipes PHP

### Combinaison `build:` + `image:` dans le compose applicatif

```yaml
api:
  build:
    context: ./simple_api
  image: student-list-api:v1
```

- `build:` permet de reproduire l'environnement avec un seul `docker compose up -d`
- `image:` nomme correctement l'image générée pour pouvoir ensuite la tagger et la pousser sur le registry privé

### Réseau dédié `pozos_network`

Isolation explicite : seuls les services attachés à ce réseau peuvent communiquer entre eux. Évite l'utilisation du réseau `default` créé implicitement par Compose, qui est moins explicite à la lecture.

### Registry dans un stack séparé

Le registry est de l'**infrastructure**, pas de l'**applicatif**. En production typique :
- 1 registry centralisé sur une machine dédiée
- N applications le consomment

Coupler les deux dans un seul compose mélangerait deux préoccupations distinctes. La séparation permet de redéployer l'application sans toucher au registry, et inversement.

### Ports `5050` et `8085` pour le registry

Pour éviter les conflits avec le stack applicatif qui utilise déjà `5000` (API) et `8080` (website). La communication interne entre `registry-ui` et `registry` reste sur le port 5000 du conteneur — seul le mapping côté hôte change.

### `REGISTRY_STORAGE_DELETE_ENABLED` et `DELETE_IMAGES`

Activés pour permettre la suppression d'images via l'UI Joxit. Pratique pour nettoyer les anciennes versions et garder le registry propre.

### Volume nommé `registry_data`

Les images poussées sur le registry sont stockées dans `/var/lib/registry` à l'intérieur du conteneur. Sans volume, un `docker compose down` ferait perdre toutes les images. Le volume nommé garantit la persistance entre redémarrages.

---

## 🐛 Problèmes rencontrés

### 1. Compilation de `python-ldap` qui échoue au build

**Symptôme** :

```
error: command 'gcc' failed: No such file or directory
ERROR: Failed building wheel for python-ldap
```

**Cause** : `python:3.13-slim` ne contient pas de compilateur C, mais `python-ldap` doit compiler du code C natif au moment de l'installation `pip`.

**Solution** : ajouter `build-essential` aux paquets installés via `apt-get` dans le Dockerfile, ainsi que les headers de développement nécessaires (`python3-dev`, `libsasl2-dev`, `libldap2-dev`, `libssl-dev`).

### 2. Versions des dépendances Python incompatibles avec Python 3.13

**Symptôme** : Le `requirements.txt` initial pointait vers `flask==2.0.0` et `Werkzeug==2.0.0` — versions trop anciennes pour fonctionner avec Python 3.13 (erreurs d'import sur des modules de la stdlib supprimés depuis).

**Solution** : monter les versions à `flask>=3.0`, `Werkzeug>=3.0`, `python-dotenv>=1.0`, `flask-httpauth>=4.8` pour rester compatible avec Python 3.13.

### 3. Conflit de ports entre le stack app et le stack registry

**Symptôme** :

```
Error: bind: address already in use
```

**Cause** : Les deux composes utilisaient initialement les mêmes ports (`5000` et `8080`).

**Solution** : déplacer le registry sur le port `5050` et son UI sur `8085`. La variable `NGINX_PROXY_PASS_URL` reste en `http://registry:5000` car la communication interne se fait sur le port du conteneur, pas celui de l'hôte.

### 4. Conteneur API en statut `Created` sans démarrer

**Symptôme** : `docker compose ps` montrait l'API en `Created` au lieu de `Up`, sans message d'erreur visible immédiatement.

**Cause** : un précédent test avec `docker run -p 80:80` avait gardé un conteneur réservant le port. Au build suivant, le bind échouait silencieusement avec un statut `Created`.

**Solution** : `docker inspect <container> --format '{{.State.Error}}'` a révélé l'erreur `bind: address already in use`. Suppression du conteneur orphelin avec `docker rm -f <container>` puis redémarrage propre.

---

## 🧹 Nettoyage

Pour arrêter et supprimer toute l'infrastructure :

```bash
# Stack applicatif
cd student-list
docker compose down

# Stack registry (avec suppression du volume = perte des images poussées)
cd registry
docker compose down -v
```

Pour supprimer aussi les images locales :

```bash
docker rmi student-list-api:v1 localhost:5050/student-list-api:v1
```

---

## 📜 Licence et crédits

POC réalisé dans le cadre d'un examen pratique Docker — formation **Eazytraining**.

Application `student_list` fournie par POZOS.
