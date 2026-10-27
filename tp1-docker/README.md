# TP1 – Docker

Application en 3 conteneurs, orchestrée avec Docker Compose :

```
Client ──► httpd (Apache, reverse proxy, port 80) ──► backend (API Spring Boot) ──► database (PostgreSQL)
```

| Dossier | Contenu |
|---|---|
| `Database/` | Image PostgreSQL + scripts SQL d'initialisation |
| `Backend/` | API Spring Boot (build multi-étapes) |
| `HttpServer/` | Apache, reverse proxy vers le backend |
| `Java-HelloWorld/`, `simpleapi/` | Premiers exercices (Java simple, puis API minimale) |
| `docker-compose.yml` | Lance toute l'application |

**Lancer l'application** : copier `.env.example` en `.env`, puis :

```bash
docker compose up -d --build
```

---

## Réponses aux questions

### 1-1 Pourquoi passer les variables d'environnement avec `-e` plutôt que dans le Dockerfile ?

- **Sécurité** : le Dockerfile est versionné, et une variable `ENV` reste gravée dans l'image. Avec `-e`, le mot de passe n'est ni dans le code, ni dans l'image.
- **Flexibilité** : la même image sert en développement, en test et en production, avec des identifiants différents.

### 1-2 Pourquoi attacher un volume au conteneur PostgreSQL ?

Un conteneur est **éphémère** : ses données disparaissent quand on le supprime. Le volume `db-data` stocke les données **en dehors du conteneur**, elles survivent donc à sa suppression. Je l'ai vérifié : un étudiant ajouté est toujours présent après avoir supprimé et recréé le conteneur.

### 1-3 Conteneur de base de données : Dockerfile et commandes

```dockerfile
FROM postgres:17.2-alpine                     # image officielle, version figée, légère
COPY scripts/ /docker-entrypoint-initdb.d/    # scripts SQL lancés au 1er démarrage (base vide)
```

| Commande | Rôle |
|---|---|
| `docker network create app-network` | Réseau où les conteneurs se joignent par leur nom |
| `docker build -t willemma/database .` | Construit l'image |
| `docker run -d --name database --network app-network -e POSTGRES_PASSWORD=... -v db-data:/var/lib/postgresql/data willemma/database` | Lance la base avec ses identifiants et son volume |
| `docker run -d -p 8090:8080 --network app-network adminer` | Interface web Adminer pour consulter la base |

### 1-4 Pourquoi un build multi-étapes ?

Pour **compiler**, il faut un JDK et Maven, alors que pour **exécuter**, un simple JRE suffit.
- **Étape 1 (build)** : image JDK + Maven, qui copie le code et lance `mvn package`.
- **Étape 2 (run)** : image JRE seule, qui récupère uniquement le `.jar` et le lance.

L'image finale est donc **plus légère et plus sûre** : elle ne contient ni code source ni outils de build.

### 1-5 Pourquoi un reverse proxy ?

C'est l'**unique point d'entrée** de l'application : seul le port 80 est ouvert. Le backend et la base ne sont pas exposés directement. Le reverse proxy permet aussi de gérer le HTTPS et la répartition de charge à un seul endroit.

### 1-6 Pourquoi Docker Compose est-il important ?

Toute l'application (conteneurs, réseaux, volumes, variables, ordre de démarrage) est décrite dans **un seul fichier versionné**, et se lance avec **une seule commande**. L'environnement est le même sur toutes les machines.

### 1-7 Commandes Docker Compose essentielles

| Commande | Rôle |
|---|---|
| `docker compose up -d --build` | Construit et démarre tout en arrière-plan |
| `docker compose ps` | État des services |
| `docker compose logs -f backend` | Logs d'un service en continu |
| `docker compose restart backend` | Redémarre un service |
| `docker compose down` | Arrête et supprime les conteneurs (les volumes sont conservés) |
| `docker compose down -v` | Idem, **et supprime les volumes** (perte des données) |
| `docker compose --profile tools up -d` | Démarre aussi Adminer |

### 1-8 Le fichier docker-compose

Les identifiants sont dans un fichier `.env`, **non versionné** (voir `.env.example`), et lus avec `${...}`.

| Choix | Intérêt |
|---|---|
| `restart: unless-stopped` | Relance automatique d'un conteneur qui plante |
| `healthcheck` (base et backend) | Docker sait si un service est réellement **prêt**, pas seulement démarré |
| `depends_on` + `service_healthy` | Le backend attend la base, Apache attend le backend |
| 2 réseaux : `front-network` / `back-network` | Apache ne peut pas joindre la base directement |
| `back-network` en `internal: true` | La base n'a aucun accès vers l'extérieur |
| Pas de `ports` sur le backend et la base | Seul le port 80 est ouvert |
| Volume `db-data` | Les données survivent à `docker compose down` |
| Adminer dans le profil `tools` | L'outil d'administration ne démarre qu'à la demande |
