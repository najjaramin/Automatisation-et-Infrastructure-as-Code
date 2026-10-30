# Projet Ansible : Déploiement de Jenkins, Docker, Portainer et d'un conteneur web

Ce projet (TP4 : Bases d'Ansible) automatise, sur un parc de machines Ubuntu, le déploiement d'un serveur web Apache, de Jenkins, de Docker, de Portainer et d'un conteneur applicatif Nginx. Il passe d'un playbook monolithique à une organisation en **rôles**.

## Architecture du projet

```text
demo1/
├── README.md
├── ansible.cfg
├── inventory.txt
├── index.html
├── Dockerfile
├── docker-compose.yml
├── webserver_install.yml
├── containers_install.yml
├── site.yml
├── captures/
│   ├── ping-pong.png
│   ├── playbook-run.png
│   ├── idempotence.png
│   ├── ssh-keys.png
│   ├── containers.png
│   └── roles-run.png
└── roles/
    ├── jenkins_install/
    │   └── tasks/main.yml
    ├── docker_portainer_install/
    │   └── tasks/main.yml
    └── container_deployment/
        └── tasks/main.yml
```

## Prérequis

- Un nœud de contrôle sous Ubuntu avec Ansible installé :
  ```bash
  sudo apt update
  sudo apt install -y ansible sshpass
  ```
- Deux nœuds gérés (Ubuntu) accessibles en SSH, avec Python 3 et un utilisateur `devops` disposant des droits sudo (idéalement sans mot de passe).
- Authentification SSH par clé configurée :
  ```bash
  ssh-keygen -t ed25519 -C "ansible-control"
  ssh-copy-id devops@192.168.30.176
  ssh-copy-id devops@192.168.30.179
  ```

## Configuration

### Inventaire (`inventory.txt`)

```ini
[webserver]
192.168.30.176

[ftpserver]
192.168.30.179

[nodes:children]
webserver
ftpserver

[nodes:vars]
ansible_user=devops
```

Vérification de l'inventaire :

```bash
ansible-inventory -i inventory.txt --graph
```

### Configuration Ansible (`ansible.cfg`)

```ini
[defaults]
inventory = ./inventory.txt
remote_user = devops
host_key_checking = False
deprecation_warnings = False
```

> `host_key_checking = False` est réservé au lab.

## Description des playbooks

### `webserver_install.yml`

Installe Apache sur le groupe `webserver`, copie `index.html` dans `/var/www/html/` et s'assure que le service est démarré et activé au boot. Ce playbook est idempotent.

```bash
ansible-playbook webserver_install.yml --syntax-check
ansible-playbook webserver_install.yml -K --check --diff
ansible-playbook webserver_install.yml -K
curl http://192.168.30.176
```

### `containers_install.yml`

Playbook monolithique qui installe Jenkins, Docker, Portainer et lance un conteneur Nginx via Docker Compose.

### `site.yml` (approche par rôles)

Playbook principal, réorganisé en rôles pour plus de modularité et de réutilisabilité :

| Rôle | Contenu |
|------|---------|
| `jenkins_install` | Ajout du dépôt, installation et démarrage de Jenkins |
| `docker_portainer_install` | Installation de Docker et Docker Compose, configuration de l'utilisateur |
| `container_deployment` | Déploiement de la stack Docker Compose (site web + Portainer) dans `/opt/demo` |

```bash
ansible-playbook site.yml
```

## Déploiement applicatif

- **`Dockerfile`** : crée une image Nginx contenant le site web (`index.html`). L'image est construite directement sur chaque nœud (`docker build`), sans passage par Docker Hub.
- **`docker-compose.yml`** : définit deux services :
  - `web` : le site web construit depuis le Dockerfile local, exposé sur le port `8081`.
  - `portainer` : l'interface de gestion de Docker, exposée sur le port `9443`.

## Accès aux services

| Service | URL |
|---------|-----|
| Apache (playbook web) | `http://192.168.30.176` |
| Conteneur web (Nginx) | `http://<IP_noeud>:8081` |
| Portainer | `https://<IP_noeud>:9443` |
| Jenkins | `http://<IP_noeud>:8080` |

Mot de passe initial de Jenkins, à lire sur le nœud concerné :

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## Idempotence

Chaque playbook doit pouvoir être exécuté plusieurs fois sans effet de bord. Le second passage de `webserver_install.yml` **et** de `site.yml` doit afficher `changed=0` dans le `PLAY RECAP`.

```bash
ansible-playbook site.yml    # 1er passage : changements attendus
ansible-playbook site.yml    # 2e passage  : changed=0
```

## Commandes utiles

```bash
# Tester la connexion avec les nœuds
ansible all -m ping

# Vérifier la syntaxe d'un playbook
ansible-playbook site.yml --syntax-check

# Afficher la configuration Ansible modifiée
ansible-config dump --only-changed

# Lancer le playbook principal
ansible-playbook site.yml
```

Avec la clé SSH configurée, `-k` n'est plus nécessaire. Ajoutez `-K` si sudo demande encore un mot de passe.

## Livrables

Captures déposées dans `captures/` :

| Fichier | Contenu |
|---------|---------|
| `ping-pong.png` | `ansible all -m ping` avec SUCCESS / pong sur les 2 nœuds |
| `playbook-run.png` | Premier passage de `webserver_install.yml` + `curl` sur le nœud web |
| `idempotence.png` | Deuxième passage avec `changed=0` dans le PLAY RECAP |
| `ssh-keys.png` | Exécution sans `-k` après `ssh-copy-id` |
| `containers.png` | Jenkins, Portainer et le conteneur web accessibles depuis le navigateur |
| `roles-run.png` | Exécution du playbook à base de rôles |
