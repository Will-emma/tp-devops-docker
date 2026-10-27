# TP DevOps

Dépôt regroupant les TP du module DevOps.

| TP | Sujet | Où ? |
|---|---|---|
| **TP1** | Docker : conteneuriser une application en 3 niveaux | [`tp1-docker/`](tp1-docker/README.md) |
| **TP2** | GitHub Actions : CI/CD et Quality Gate SonarCloud | ce README + racine du dépôt |
| **TP3** | Ansible : déployer l'application sur un serveur | [`tp3-ansible/`](tp3-ansible/README.md) |

---

# TP2 – GitHub Actions

À chaque push, deux workflows s'enchaînent automatiquement :

```
push ──► CI (main.yml)                      ──succès sur main──►  CD (build-and-push.yml)
         test-backend :                                           build-and-push-docker-image :
         compile + tests + SonarCloud                             construit et publie les 3 images
         (branches main et develop)                               sur Docker Hub (main uniquement)
```

| Fichier / dossier | Rôle |
|---|---|
| `.github/workflows/main.yml` | **CI** : build, tests unitaires et d'intégration, analyse SonarCloud |
| `.github/workflows/build-and-push.yml` | **CD** : build et push des images Docker |
| `simple-api/`, `database/`, `http-server/` | Les 3 applications, chacune avec son `Dockerfile` |

**Résultats**

- Images publiées : `willemma/tp-devops-simple-api`, `willemma/tp-devops-database`, `willemma/tp-devops-httpd`
- SonarCloud : Quality Gate **Passed**, couverture de tests **92,1 %**

---

## Réponses aux questions

### Que fait `mvn clean verify` ?

`clean` supprime les anciens builds. `verify` compile tout, puis lance les **tests unitaires** et les **tests d'intégration**.

### Tests unitaires et tests de composants ?

- **Unitaire** : teste une seule classe isolée, sans base de données (`*ServiceTest`).
- **Intégration (composants)** : teste plusieurs couches ensemble, jusqu'à une vraie base de données (`*ControllerTestIT`).

### 2-1 Que sont les Testcontainers ?

Des bibliothèques Java qui lancent de **vrais conteneurs Docker pendant les tests** (ici un PostgreSQL), puis les suppriment à la fin. Les tests d'intégration utilisent donc une vraie base, jetable, sans rien installer.

### 2-2 Pourquoi des variables sécurisées ?

Pour ne **jamais écrire d'identifiants dans le code** : tout le monde pourrait les lire, et l'historique Git les garderait pour toujours. Les secrets GitHub (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `SONAR_TOKEN`) sont chiffrés, masqués (`***`) dans les logs, et modifiables sans toucher au code.

### 2-3 Pourquoi `needs: test-backend` ?

Sans `needs`, les deux jobs démarrent **en même temps** : on pourrait publier une image dont les tests échouent. Avec `needs`, les images ne sont construites que si les tests passent.

Depuis la séparation des pipelines, c'est `on: workflow_run` + `if: conclusion == 'success'` qui joue ce rôle entre les deux fichiers.

### 2-4 Pourquoi pousser les images Docker ?

Pour les stocker dans un **registre accessible de partout** : un serveur (comme au TP3 avec Ansible) ou un collègue récupère exactement l'image testée avec un simple `docker pull`, sans recompiler.

---

## Quality Gate (SonarCloud)

SonarCloud analyse le code à chaque push : bugs, failles de sécurité, code mal écrit, duplication et couverture de tests. Il rend ensuite un verdict, la **Quality Gate** (*Passed* ou *Failed*).

- **Analyse automatique désactivée** dans SonarCloud : sinon, elle entre en conflit avec celle lancée par la CI.
- **Couverture** : le rapport JaCoCo est généré à la phase `verify`, pour compter aussi les tests d'intégration (53,6 % → 92,1 %).
- La Quality Gate « Sonar way » ne juge que le **nouveau code** (*Clean as You Code*) : on empêche que la qualité se dégrade, sans obliger à tout corriger d'un coup.

## Séparation des pipelines

| Workflow | Quand ? |
|---|---|
| **CI** (`main.yml`) | push sur `main` et `develop`, Pull Requests |
| **CD** (`build-and-push.yml`) | après la CI, **uniquement sur `main`** et **uniquement si elle a réussi** (`workflow_run`) |

- `needs:` ne fonctionne qu'entre deux jobs d'un même fichier. Entre deux fichiers, on utilise `on: workflow_run`.
- `ref: ${{ github.event.workflow_run.head_sha }}` : le CD construit exactement le commit qui a été testé.

## Branche `main` protégée

Pas de push direct sur `main` : tout passe par une **Pull Request** depuis `develop`. La PR n'est fusionnable que si la CI (`test-backend`) est verte. Le force-push et la suppression de `main` sont interdits.
