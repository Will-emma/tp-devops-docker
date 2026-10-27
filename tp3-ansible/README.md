# TP3 – Ansible

Ansible déploie automatiquement l'application du TP1 sur un serveur distant : il installe Docker, puis lance les 3 conteneurs à partir des images publiées sur Docker Hub par le TP2.

```
poste local ──(SSH)──► serveur  →  installe Docker  →  crée les réseaux  →  lance database, backend, httpd
```

| Fichier | Rôle |
|---|---|
| `inventories/setup.yml` | Liste des serveurs, et comment s'y connecter |
| `playbook.yml` | Ordre des rôles à appliquer |
| `roles/` | Un rôle par tâche : `docker`, `network`, `database`, `app`, `proxy` |

**Lancer le déploiement :**

```bash
ansible-playbook -i inventories/setup.yml playbook.yml
```

---

## Réponses aux questions

### 3-1 L'inventaire et les commandes de base

```yaml
all:
  vars:
    ansible_user: admin                                  # utilisateur SSH sur le serveur
    ansible_ssh_private_key_file: /home/will/.ssh/id_rsa # clé privée pour se connecter
    postgres_db: db                                      # variables utilisées par les rôles
    postgres_user: usr
    postgres_password: pwd
  children:
    prod:                                                # groupe de serveurs "prod"
      hosts: willemmanuel.poaty.takima.school
```

L'inventaire indique **sur quels serveurs** agir (regroupés, ici dans `prod`) et **comment s'y connecter** (utilisateur et clé SSH). Les variables définies sous `vars` sont disponibles dans tous les rôles.

| Commande | Rôle |
|---|---|
| `ansible all -i inventories/setup.yml -m ping` | Teste la connexion SSH à tous les serveurs |
| `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` | Récupère des informations sur le serveur (ici l'OS) |
| `ansible-galaxy init roles/docker` | Crée la structure d'un rôle |
| `ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check` | Vérifie la syntaxe du playbook |
| `ansible-playbook -i inventories/setup.yml playbook.yml` | Lance le déploiement |

> Le mot de passe de la base est en clair dans l'inventaire : c'est une valeur de TP. En vrai, on le chiffrerait avec **Ansible Vault**.

### 3-2 Le playbook

```yaml
- hosts: all          # s'applique à tous les serveurs de l'inventaire
  gather_facts: true  # récupère les infos du serveur (OS, version…)
  become: true        # exécute en root (sudo), nécessaire pour installer des paquets
  roles:
    - docker          # 1. installe Docker
    - network         # 2. crée les réseaux
    - database        # 3. lance la base
    - app             # 4. lance le backend
    - proxy           # 5. lance Apache
```

- **Pourquoi des rôles ?** Chaque rôle a une seule responsabilité. Le playbook reste court et lisible, et un rôle peut être réutilisé dans un autre projet.
- **Pourquoi cet ordre ?** Il respecte les dépendances : Docker avant les conteneurs, les réseaux avant les conteneurs qui s'y branchent, la base avant le backend.
- **`ansible_python_interpreter: /opt/docker_venv/bin/python`** : les modules Docker d'Ansible ont besoin du SDK Python `docker`. Le rôle `docker` l'installe dans un environnement virtuel, et on indique aux autres rôles d'utiliser ce Python-là.

Le rôle `docker` installe les prérequis, ajoute le dépôt officiel de Docker, installe `docker-ce`, crée l'environnement Python avec le SDK `docker`, et vérifie que le service Docker tourne.

### 3-3 Les tâches `docker_container`

Exemple avec le backend (rôle `app`) :

```yaml
- name: Run backend
  community.docker.docker_container:
    name: backend                     # nom du conteneur = son adresse sur le réseau
    image: willemma/backend           # image publiée sur Docker Hub
    pull: true                        # récupère toujours la dernière version de l'image
    restart_policy: unless-stopped    # relance automatique en cas de plantage
    env:                              # variables lues dans l'inventaire
      DB_HOST: database
      DB_NAME: "{{ postgres_db }}"
      DB_USER: "{{ postgres_user }}"
      DB_PASSWORD: "{{ postgres_password }}"
    networks:
      - name: front-network           # joignable par Apache
      - name: back-network            # peut joindre la base
```

| Rôle | Conteneur | Points clés |
|---|---|---|
| `network` | — | Crée `front-network`, et `back-network` en `internal: true` (aucun accès vers l'extérieur) |
| `database` | `database` | Volume `db-data` pour la persistance, branché seulement sur `back-network` |
| `app` | `backend` | Branché sur les 2 réseaux : il fait le lien entre Apache et la base |
| `proxy` | `httpd` | **Seul conteneur exposé** (`80:80`), branché seulement sur `front-network` |

On retrouve la même architecture qu'avec Docker Compose au TP1, mais déployée à distance et de manière **reproductible** : on peut relancer le playbook autant de fois qu'on veut, Ansible ne change que ce qui doit l'être.

### Bonus

À venir.
