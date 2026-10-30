# TP5 : Lab reproductible avec Vagrant et Ansible

Ce projet déploie un lab de deux machines virtuelles Ubuntu avec **Vagrant** (provider VirtualBox). Chaque VM est ensuite configurée par **Ansible** : installation de Docker, déploiement de Portainer et lancement d'un conteneur web via `docker-compose.yml`.

## Architecture

Les deux VM utilisent la box `bento/ubuntu-24.04`.

| VM | IP privée | Web (hôte → invité) | Portainer (hôte → invité) |
|----|-----------|---------------------|---------------------------|
| `lab1` | 192.168.56.11 | 8081 → 8080 | 9001 → 9000 |
| `lab2` | 192.168.56.12 | 8091 → 8080 | 9011 → 9000 |

Les ports « hôte » sont ceux de la VM de lab (celle qui exécute Vagrant). Chemin d'une requête : navigateur du PC physique → port redirigé de la VM de lab → port de la VM invitée Vagrant → conteneur Docker.

## Structure du projet

```text
lab/
├── Vagrantfile          # Définition des 2 VM, réseau privé, redirections de ports
├── playbook.yml         # Playbook Ansible : Docker, Portainer, conteneur web
├── docker-compose.yml   # Conteneur de l'application web
├── ansible.cfg          # (optionnel) réglages Ansible
├── roles/               # (optionnel) rôles réutilisés du TP4
├── captures/            # Captures du TP
└── README.md
```

## Prérequis

Sur la VM de lab (Ubuntu) :

- Virtualisation imbriquée activée (sous Hyper-V : `Set-VMProcessor -VMName <VM> -ExposeVirtualizationExtensions $true`, VM arrêtée). Vérification : `grep -Ec '(vmx|svm)' /proc/cpuinfo` doit renvoyer une valeur supérieure à 0.
- VirtualBox, Vagrant (dépôt HashiCorp) et Ansible installés :
  ```bash
  vagrant --version
  VBoxManage --version
  ansible --version
  ```
- Ressources : 8 Go de RAM et 4 vCPU recommandés pour la VM de lab, au moins 15 Go de disque libres. Avec moins de RAM, démarrez et provisionnez **une VM à la fois**, sinon les VM invitées peuvent passer en `aborted`.

## Déploiement

```bash
cd ~/lab
vagrant validate
vagrant up lab1 --no-provision      # crée la VM sans lancer Ansible
vagrant provision lab1              # lance Ansible sur lab1
vagrant up lab2 --no-provision
vagrant provision lab2
vagrant status                      # les 2 VM doivent être "running"
```

Avec assez de ressources, `vagrant up` crée et provisionne les deux VM en une fois.

## Accès aux services

Depuis la VM de lab, en remplaçant `IP-VM-LAB` par son adresse (`ip -br a`) :

| Service | lab1 | lab2 |
|---------|------|------|
| Application web | `http://IP-VM-LAB:8081` | `http://IP-VM-LAB:8091` |
| Portainer | `http://IP-VM-LAB:9001` | `http://IP-VM-LAB:9011` |

Test direct par le réseau privé :

```bash
curl http://192.168.56.11:8080
curl http://192.168.56.12:8080
```

### Première connexion à Portainer

Portainer se verrouille si le compte administrateur n'est pas créé dans les **5 minutes** après son démarrage. Il demande aussi un `setup_token`. Procédure :

```bash
vagrant ssh lab1 -c "sudo docker restart portainer"
sleep 5
vagrant ssh lab1 -c "sudo docker logs portainer 2>&1 | grep setup_token | tail -1"
```

Ouvrez ensuite immédiatement `http://IP-VM-LAB:9001`, collez le token, créez l'utilisateur admin (12 caractères minimum) et choisissez l'environnement Docker local. Le dernier token affiché est le seul valide.

## Commandes utiles

```bash
vagrant status                  # état des VM
vagrant ssh lab1                # se connecter à une VM
vagrant provision lab1          # relancer Ansible
vagrant reload --provision      # redémarrer et reprovisionner
vagrant halt                    # arrêter (conserve les disques)
vagrant destroy -f              # tout supprimer
vagrant ssh lab1 -c "sudo docker ps"
```

## Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `Connection refused` sur le SSH au `vagrant up` | Port SSH forcé dans le Vagrantfile (`ssh.port`) en conflit avec la redirection automatique | Supprimer les lignes `ssh.port` et `ssh.guest_port`, puis `vagrant destroy -f` et `vagrant up` |
| VM `aborted`, SSH perdu pendant Ansible | Manque de RAM ou de CPU sur la VM de lab | Augmenter les ressources, ou travailler une VM à la fois |
| `VERR_DISK_FULL` à l'import de la box | Disque de la VM de lab plein | Libérer de l'espace (`vagrant destroy -f`, `VBoxManage unregistervm <nom> --delete`, `sudo apt clean`) |
| Portainer redirige vers `/timeout.html` | Délai de 5 minutes dépassé | Redémarrer le conteneur et créer le compte aussitôt |
| `permission denied` sur `docker ps` | L'utilisateur `vagrant` n'est pas dans le groupe `docker` | Utiliser `sudo docker ps` |

## Idempotence

Un second passage d'Ansible ne doit rien modifier :

```bash
vagrant provision lab1
```

Le `PLAY RECAP` doit afficher `changed=0`.

## Livrables

Captures à déposer dans `captures/` :

| Fichier | Contenu |
|---------|---------|
| `nested-virt.png` | `grep -Ec '(vmx\|svm)' /proc/cpuinfo` supérieur à 0 et `vagrant --version` |
| `vagrant-demo.png` | Page de `vagrant_demo` ouverte depuis le PC physique (port 8080) |
| `vagrant-status.png` | `vagrant status` avec les 2 VM en `running` |
| `lab-services.png` | Services accessibles sur les 2 VM |


# TP6 : Infrastructure as Code avec Terraform (intégration Vagrant + Ansible)

Ce projet utilise **Terraform** pour l'Infrastructure as Code, couplé à **Vagrant** (création des VM) et **Ansible** (configuration). Il déploie une application web conteneurisée (CV One Page) sur deux VM Ubuntu.

## Architecture

Le workflow comporte trois étapes :

1. **Vagrant** crée 2 VM Ubuntu 24.04 (`bento/ubuntu-24.04`, provider VirtualBox).
2. **Ansible**, appelé par Vagrant, installe Docker et Terraform sur chaque VM.
3. **Terraform**, exécuté par Ansible dans chaque VM, tire l'image Docker du CV (nginx) depuis Docker Hub et lance le conteneur.

| VM | IP privée | Port du CV | Rôle |
|----|-----------|------------|------|
| `vm1` | 192.168.56.21 | 8001 | Conteneur CV (instance 1) |
| `vm2` | 192.168.56.22 | 8002 | Conteneur CV (instance 2) |

## Structure du projet

```text
tp6-integration/
├── Vagrantfile          # 2 VM : réseau privé, RAM, redirections de ports
├── playbook.yml         # Ansible : Docker + Terraform, puis terraform apply
├── main.tf              # Terraform : provider Docker, image, conteneur
├── variables.tf         # Variables (ex. port externe)
├── .terraform.lock.hcl  # À versionner
├── Dockerfile           # Image nginx du CV One Page
├── index.html / css/    # Contenu du CV
├── proxy/
│   └── nginx.conf       # (Défi) reverse proxy nginx
├── captures/
└── README.md
```

Ne versionnez pas `terraform.tfstate`, `terraform.tfstate.backup`, `.terraform/` ni les `*.tfvars` contenant des secrets.

## Prérequis

Sur la VM Ubuntu hôte (celle qui exécute Vagrant) :

- Virtualisation imbriquée activée (`grep -Ec '(vmx|svm)' /proc/cpuinfo` supérieur à 0).
- VirtualBox, Vagrant et Ansible installés. Pour le TP, Docker et Terraform sont aussi installés sur l'hôte.
- Un compte Docker Hub avec l'image du CV poussée.
- Ressources : prévoir au moins 8 Go de RAM et 4 vCPU pour l'hôte, ainsi que 15 Go de disque libres. Avec moins, démarrez et provisionnez une VM à la fois : les VM invitées peuvent passer en `aborted`, et un disque plein (`no space left on device`) empêche Terraform d'écrire son état.

## Partie 1 : Terraform en local (Docker)

```bash
mkdir -p ~/terraform/docker-nginx && cd ~/terraform/docker-nginx
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
docker ps
curl http://localhost:8001
terraform state list
terraform apply              # doit répondre : No changes
terraform apply -var external_port=8002
terraform destroy
```

Docker distant (provider via SSH, authentification par clé obligatoire) :

```hcl
provider "docker" {
  host = "ssh://user@devopslab:22"
}
```

## Partie 2 : Intégration Vagrant + Ansible + Terraform

```bash
cd ~/tp6-integration
vagrant validate
vagrant up vm1 --no-provision
vagrant provision vm1
vagrant up vm2 --no-provision
vagrant provision vm2
vagrant status               # les 2 VM doivent être "running"
```

Si l'hôte dispose d'assez de ressources, `vagrant up` crée et provisionne tout en une fois.

## Accès aux services

Depuis un navigateur, en remplaçant `IP-HOTE` par l'adresse de la VM qui exécute Vagrant (`ip -br a`) :

- Instance 1 : `http://IP-HOTE:8001`
- Instance 2 : `http://IP-HOTE:8002`

Test direct par le réseau privé :

```bash
curl http://192.168.56.21:8001
curl http://192.168.56.22:8002
```

## Défi : reverse proxy nginx

Un conteneur nginx sur l'hôte sert de point d'entrée unique et redirige vers les deux instances (adresses privées des VM) :

```bash
cd proxy
docker run -d --name reverse-proxy -p 8080:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro nginx
```

Accès via le proxy :

- `http://IP-HOTE:8080/vm1`
- `http://IP-HOTE:8080/vm2`

## Idempotence

Relancer `terraform apply` doit afficher `No changes`, et un second `vagrant provision` doit afficher `changed=0` dans le `PLAY RECAP`.

## Nettoyage

```bash
terraform destroy            # dans chaque dossier Terraform
vagrant destroy -f
docker rm -f reverse-proxy   # si le défi a été réalisé
```

## Dépannage

| Symptôme | Cause probable | Solution |
|----------|----------------|----------|
| `no space left on device` pendant `terraform apply` | Disque plein | `docker system prune -af`, `sudo apt clean`, détruire les VM inutiles, étendre le volume LVM si besoin |
| VM `aborted` ou SSH perdu | RAM ou CPU insuffisants | Augmenter les ressources, ou travailler une VM à la fois |
| `permission denied` sur `docker ps` | Utilisateur absent du groupe `docker` | `sudo usermod -aG docker $USER`, puis se reconnecter |
| Erreur de connexion au Docker distant | Clé SSH absente ou `devopslab` non résolu | `ssh-copy-id`, ajouter le nom dans `/etc/hosts` ou utiliser l'IP |

## Livrables

Captures à déposer dans `captures/` :

| Fichier | Contenu |
|---------|---------|
| `terraform-plan.png` | Sortie de `terraform plan` |
| `terraform-apply.png` | `terraform apply`, puis `docker ps` et `curl http://localhost:8001` |
| `terraform-remote.png` | Conteneur créé sur `devopslab` via le provider SSH |
| `integration-cv.png` | Le CV servi sur vm1:8001 et vm2:8002 |
| `reverse-proxy.png` | (défi) Le CV servi par le reverse proxy nginx |
