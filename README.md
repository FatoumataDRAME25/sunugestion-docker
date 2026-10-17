# SunuGestion — Docker

Ce dossier assemble toute l'application SunuGestion avec Docker : base de données, backend Django, frontend Angular et, à la demande, un tunnel ngrok. Il ne contient pas de code : il utilise les deux projets voisins.

```
Bureau/
├── SunuGestion-Docker/          ← ce dossier
│   ├── docker-compose.yml       mode normal (comme en production)
│   ├── docker-compose.dev.yml   mode développement (rechargement automatique)
│   ├── proxy.docker.json        proxy de ng serve en mode développement
│   ├── .env                     secrets (jamais publié)
│   └── sauvegardes/             copies de bases et de configuration (jamais publié)
├── Django/sunugestion-backend/  backend (Dockerfile, entrypoint.sh, .env, .env.docker)
└── Angular/sunugestion/         frontend (Dockerfile, nginx.conf)
```

Les chemins `../Django/...` et `../Angular/...` du compose supposent exactement cette organisation.

---

## Architecture

```
Navigateur ──► http://localhost:8080 ──► frontend (nginx)
                                          ├─ pages Angular
                                          ├─ /api, /ocr, /admin, /static ──► backend (Django + gunicorn) ──► postgres
                                          └─ /media (logos) ──► volume media_data
backend ──► n8n (conteneur à part, port 5678) ──► OpenRouter (IA)
```

| Service | Rôle | Adresse sur la machine |
|---|---|---|
| `postgres` | Base PostgreSQL 17 | `localhost:5433` |
| `backend` | Django (gunicorn en mode normal, runserver en développement) | `localhost:8001` (débogage) |
| `frontend` | nginx (mode normal) ou ng serve (développement) | **`localhost:8080`** : porte d'entrée |
| `ngrok` | Adresse publique fixe (profil `ngrok`, optionnel) | `https://reviving-gurgle-pauper.ngrok-free.dev` |

Tous les ports sont publiés sur `127.0.0.1` uniquement : rien n'est joignable depuis le Wi-Fi.

**Volumes** (les données survivent à la recréation des conteneurs) :

| Volume | Contenu |
|---|---|
| `postgres_data` | Base de données |
| `media_data` | Logos des groupements |
| `paddle_models` | Modèles de l'OCR (téléchargés au premier import par photo) |
| `frontend_node_modules`, `frontend_angular_cache` | Paquets npm et cache Angular du mode développement |

---

## Prérequis

- Docker avec **Docker Compose 2.24 ou plus récent** (`docker compose version`) : le mode développement utilise `!reset` et `!override`.
- Les deux projets voisins, organisés comme ci-dessus.
- **n8n** lancé à part, sur le port 5678, avec le workflow « Extraction personnes (OCR -> LLM) » actif : il sert à l'import de membres par photo.

---

## Configuration (une seule fois)

### 1. `SunuGestion-Docker/.env`

Lu automatiquement par Docker Compose. Droits conseillés : `chmod 600 .env`.

```env
POSTGRES_PASSWORD=...      # mot de passe de la base Docker
NGROK_AUTHTOKEN=...        # jeton du compte ngrok (seulement pour le profil ngrok)
```

Sans `POSTGRES_PASSWORD`, Docker refuse de démarrer avec un message explicite.

### 2. Backend : `.env` et `.env.docker`

Le backend lit `.env` (réglages communs avec le développement), puis `.env.docker` (réglages propres à Docker, prioritaires). Créer `.env.docker` à partir de `.env.docker.example` :

| Variable | Valeur Docker |
|---|---|
| `DB_HOST` / `DB_PORT` | `postgres` / `5432` |
| `DB_NAME` / `DB_USER` / `DB_PASSWORD` | `sunugestion` / `sunugestion` / **le même mot de passe que `POSTGRES_PASSWORD`** |
| `DEBUG` | `False` (le mode développement le force à `True`) |
| `N8N_WEBHOOK_URL` | `http://host.docker.internal:5678/webhook/extract-persons` |
| `NUM_PROXIES` | `2` (ngrok puis nginx ; reste correct sans ngrok) |
| `ALLOWED_HOSTS` | `localhost,127.0.0.1,backend,<adresse-ngrok>` |
| `PAYDUNYA_RETURN_URL` / `PAYDUNYA_CANCEL_URL` | `https://<adresse-ngrok>/paiement/retour` et `/paiement/annule` |

`PAYDUNYA_CALLBACK_URL` vient de `.env` et doit viser `https://<adresse-ngrok>/api/paiements/paydunya/callback/`.

### 3. Changer le mot de passe de la base

`POSTGRES_PASSWORD` n'est lu qu'à la **création** de la base. Pour le changer ensuite :

```bash
docker exec -it sunugestion-postgres psql -U sunugestion -d sunugestion \
  -c "ALTER USER sunugestion WITH PASSWORD 'nouveau_mot_de_passe';"
```

Puis mettre la même valeur dans `.env` (`POSTGRES_PASSWORD`) et dans `.env.docker` (`DB_PASSWORD`), et relancer `docker compose up -d`. Éviter `$`, `#`, les guillemets et les espaces.

---

## Utilisation

Toutes les commandes se lancent depuis ce dossier.

### Mode normal (comme en production)

```bash
docker compose up -d
```

gunicorn, nginx, `DEBUG=False`. Après une modification du code ou de `requirements.txt` :

```bash
docker compose up -d --build
```

### Mode développement (rechargement automatique)

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

Le code est lu directement dans les dossiers des projets :

| Modification | Effet |
|---|---|
| Fichier Python | runserver redémarre (1 à 2 s) |
| Fichier Angular | ng serve recompile et recharge la page (environ 5 s) |
| `requirements.txt` | relancer la commande avec `--build` |
| `package.json` | `docker compose restart frontend` |

Pour revenir au mode normal : `docker compose up -d`.

### Avec l'adresse publique (activation des membres, PayDunya)

```bash
docker compose --profile ngrok up -d
```

Lance le mode normal **et** le tunnel. Une seule adresse, fixe, sert à la fois :

- les liens d'activation (`/activation?token=...`), envoyés par WhatsApp ;
- le retour PayDunya (`/paiement/retour`, `/paiement/annule`) ;
- la notification PayDunya (`/api/paiements/paydunya/callback/`).

Arrêter seulement le tunnel : `docker compose --profile ngrok stop ngrok`.

**Règles :**

- Le tunnel vise le nginx du mode normal. Avec le mode développement, il répond 502 : `DEBUG=True` n'est jamais exposé sur internet. Ne pas lancer ngrok à la main vers le mode développement.
- Le compte ngrok gratuit n'autorise qu'**un tunnel à la fois** : ne pas lancer ngrok dans un terminal en même temps.
- Les tests d'activation et de paiement se font dans la version Docker, qui a sa propre base.
- `/admin/` est fermé à travers ngrok (404) : sa page de connexion n'a pas de limite de tentatives. L'admin s'utilise sur `http://localhost:8080/admin/`.
- Le tunnel ne redémarre pas après un redémarrage de l'ordinateur : l'application ne redevient pas publique d'elle-même.

### Commandes utiles

```bash
docker compose ps                    # état des conteneurs
docker compose logs -f               # journaux en direct (Ctrl+C quitte l'affichage seulement)
docker compose logs -f backend       # un seul service
docker compose stop                  # arrêter (données conservées)
docker compose start                 # redémarrer
```

Les conteneurs tournent en arrière-plan (`-d`) : rien ne s'affiche dans le terminal, c'est normal. Les journaux se lisent avec `docker compose logs`.

---

## Données

La base Docker est **séparée** de la base de développement (celle de `runserver` sur le port 8000). Ce qui est fait sur `localhost:4200` n'apparaît pas sur `localhost:8080`, et inversement.

### Sauvegarder la base Docker

```bash
docker exec sunugestion-postgres pg_dump -U sunugestion -d sunugestion --no-owner --no-privileges \
  > sauvegardes/docker-$(date +%Y%m%d-%H%M).sql
chmod 600 sauvegardes/*
```

### Copier la base de développement dans Docker

Remplace tout le contenu de la base Docker.

```bash
# 1. Copie de la base de développement (identifiants du .env du backend)
pg_dump -h localhost -p 5432 -U <DB_USER> -d <DB_NAME> --no-owner --no-privileges \
  > sauvegardes/dev-$(date +%Y%m%d-%H%M).sql

# 2. Arrêt du backend Docker, vidage puis restauration
docker compose stop backend
docker exec sunugestion-postgres psql -U sunugestion -d sunugestion \
  -c "DROP SCHEMA public CASCADE; CREATE SCHEMA public;"
docker exec -i sunugestion-postgres psql -U sunugestion -d sunugestion -v ON_ERROR_STOP=1 \
  < sauvegardes/dev-AAAAMMJJ-HHMM.sql

# 3. Redémarrage (les migrations manquantes s'appliquent au démarrage)
docker compose up -d
```

Les identifiants de connexion de l'application (téléphone et PIN) sont alors les mêmes qu'en développement.

---

## Tâches planifiées

Trois commandes de maintenance doivent tourner chaque jour : `traiter_abonnements`, `traiter_retards`, `supprimer_invitations_expirees`. Elles créent des notifications dans l'application (pas d'e-mail ni de SMS) et suppriment les comptes jamais activés.

Sur la machine de développement, deux **timers systemd** (utilisateur, sans `sudo`) les lancent chaque jour à 1 h. Si la machine est en veille ou éteinte, c'est **rattrapé au réveil ou au démarrage**, ce que cron ne fait pas.

| Timer | Environnement |
|---|---|
| `sunugestion-taches-docker` | `docker exec sunugestion-backend ...` (échoue sans conséquence si les conteneurs sont arrêtés) |
| `sunugestion-taches-dev` | venv et base de développement |

Fichiers dans `~/.config/systemd/user/`.

```bash
systemctl --user list-timers                              # prochaines exécutions
journalctl --user -u sunugestion-taches-docker -n 30      # résultat de la dernière exécution
systemctl --user start sunugestion-taches-docker.service  # lancer maintenant (tâches réelles)
systemctl --user disable --now sunugestion-taches-docker.timer   # désactiver
```

Voir les comptes qui seraient supprimés, sans rien supprimer :

```bash
docker exec sunugestion-backend python manage.py supprimer_invitations_expirees --simulation
```

Sur un serveur (allumé en permanence), un cron classique suffit :

```
0 1 * * * docker exec sunugestion-backend python manage.py traiter_abonnements
0 1 * * * docker exec sunugestion-backend python manage.py traiter_retards
0 1 * * * docker exec sunugestion-backend python manage.py supprimer_invitations_expirees
```

---

## Dépannage

| Symptôme | Cause et solution |
|---|---|
| L'import par photo échoue avec « n8n a répondu une erreur » | Après un changement de Wi-Fi, n8n garde l'ancien serveur DNS : `docker restart n8n`. |
| Le premier import par photo prend plusieurs minutes | Téléchargement des modèles de l'OCR, une seule fois (conservés dans `paddle_models`). Ensuite, environ 5 s. |
| L'adresse ngrok répond 502 | Le mode développement tourne : lancer le mode normal avec le profil ngrok. |
| L'adresse ngrok répond 404 sur toutes les pages | Le tunnel n'est pas lancé. |
| La base semble vide après avoir déplacé ce dossier | Les volumes portent le nom du dossier (`sunugestion-docker_...`). Garder le nom du dossier, ou ajouter `name: sunugestion-docker` en tête du compose. |
| « port is already allocated » | Un autre programme utilise 8080, 8001 ou 5433 : `ss -ltnp \| grep 8080`. |
| Première construction très longue | Le backend télécharge environ 450 Mo de paquets (PaddlePaddle, OpenCV). Les constructions suivantes réutilisent le cache. |
| Rien ne s'affiche dans le terminal | Normal avec `-d` : `docker compose logs -f`. |

---

## Sécurité

- Ne jamais publier `.env`, `.env.docker` ni `sauvegardes/` : ils contiennent des secrets et des données réelles (voir `.gitignore`).
- Les ports sont limités à `127.0.0.1`. Ne pas les rouvrir sur `0.0.0.0`.
- Avec ngrok, l'application est publique tant que le tunnel tourne : l'arrêter quand il ne sert plus.

### À faire avant un vrai déploiement

- Une `SECRET_KEY` et un mot de passe de base propres au serveur.
- Remplacer l'adresse ngrok par le vrai domaine : `URL_PUBLIQUE` (frontend), adresses PayDunya, `ALLOWED_HOSTS`, `CSRF_TRUSTED_ORIGINS` ; revoir `NUM_PROXIES` selon l'installation.
- Protéger le webhook n8n par une clé partagée.
- Faire tourner le backend avec un utilisateur non-root.
- Ajouter un en-tête `Content-Security-Policy` dans nginx.
