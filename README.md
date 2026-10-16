# hanzub

Playbook Ansible (`install-tools.yml`) qui installe les outils de développement Kubernetes et
les utilitaires du quotidien sur Ubuntu/Debian.

## Utilisation

### En production (automatique)
Le playbook est lancé par `ansible-pull` via un service systemd one-shot au premier
démarrage du master Ubuntu, fraîchement installé avec un `autoinstall.yml`.

### Manuellement (test)
Installer Ansible, puis lancer le playbook sur la machine locale :

```bash
sudo apt update && sudo apt install -y ansible
ansible-playbook -i localhost, -c local -K install-tools.yml
```

### Installation minimale
Le tag `minimal` limite l'installation au socle (00), à la bureautique (02), à un sous-ensemble
des outils du quotidien (03 : `nano`, `zsh`, `git`, `wget`, `unzip`, `tar`, `bat`, `gufw`,
`deja-dup`, `gnome-tweaks`, `smartmontools`), aux secrets git (06), au pare-feu (07) et à zsh (08). Sont
exclus : Kubernetes (01), Docker (05) et le reste de la liste 03. Les vérifications de départ
(APT, architecture) s'exécutent toujours.

```bash
ansible-playbook -i localhost, -c local -K install-tools.yml --tags minimal
ansible-pull … install-tools.yml --tags minimal   # en production
```

Sans `--tags`, l'installation est complète. `--list-tasks --tags minimal` affiche le périmètre.

## Organisation

`install-tools.yml` est un orchestrateur : il porte les variables et les vérifications, puis
importe une liste de tâches par spécification. Chaque spec de `doc/specs/` correspond à un
fichier de `tasks/` de même numéro et de même nom, ce qui permet de suivre l'avancement.

| Spec (`doc/specs/`)          | Tâches (`tasks/`)            | Statut        |
|------------------------------|------------------------------|---------------|
| *(socle, sans spec)*         | `00-prerequis.yml`           | en place      |
| `01-outils-kubernetes.md`    | `01-outils-kubernetes.yml`   | implémentée   |
| `02-bureautique.md`          | `02-bureautique.yml`         | implémentée (à tester) |
| `03-outils-quotidiens.md`    | `03-outils-quotidiens.yml`   | implémentée   |
| `05-docker.md`               | `05-docker.yml`              | implémentée (à tester) |
| `06-secrets-git.md`          | `06-secrets-git.yml`         | implémentée (à tester) |
| `07-pare-feu.md`             | `07-pare-feu.yml`            | implémentée (à tester) |
| `08-zsh.md`                  | `08-zsh.yml`                 | implémentée (à tester) |

Pour ajouter un outil : rédiger ou compléter la spec, puis coder la liste de tâches
correspondante. Ne pas ajouter de tâche directement dans `install-tools.yml`.
Cette organisation reste compatible avec `ansible-pull`, qui exécute le playbook depuis
la copie du dépôt.

## Ce qui est installé

Le playbook vérifie d'abord que le système est basé sur APT (Debian/Ubuntu) et que
l'architecture est `amd64` ou `arm64`.

### Prérequis APT
`apt-transport-https`, `ca-certificates`, `curl`, `gnupg`

### Dépôts APT ajoutés
- Kubernetes (`pkgs.k8s.io`, branche `v1.34`)
- Helm (`packages.buildkite.com`)

### Outils Kubernetes
| Outil      | Version   | Source                 |
|------------|-----------|------------------------|
| `kubectl`  | v1.34.x   | dépôt APT Kubernetes   |
| `helm`     | dernière  | dépôt APT Helm         |
| `k9s`      | v0.50.9   | release GitHub         |
| `minikube` | v1.37.0   | binaire Google Storage |

`k9s` et `minikube` sont installés dans `/usr/local/bin`.

### Utilitaires système et quotidiens (APT)
| Paquet      | Usage                                   |
|-------------|-----------------------------------------|
| `nano`      | éditeur de texte                        |
| `zsh`       | shell alternatif                        |
| `git`       | gestionnaire de versions                |
| `btop`      | utilisation CPU / RAM                   |
| `htop`      | utilisation CPU / RAM                   |
| `ncdu`      | utilisation du disque                   |
| `jq`        | manipulation JSON                       |
| `yq`        | manipulation YAML                       |
| `ydiff`     | comparaison de fichiers                 |
| `bat`       | visualisation de fichiers (`less` amélioré) |
| `wget`      | téléchargement                          |
| `unzip`     | décompression                           |
| `tar`       | archivage                               |
| `musl`      | compilation de binaires statiques       |
| `virtualbox` | hyperviseur de machines virtuelles     |
| `virtualbox-guest-additions-iso` | ISO des Guest Additions |
| `virtualbox-ext-pack` | Extension Pack VirtualBox (USB 2/3, RDP, PXE, NVMe, chiffrement de disque) |
| `gufw` | interface graphique du pare-feu (ufw) |
| `remmina` | client de bureau à distance |
| `remmina-plugin-rdp` | greffon RDP pour Remmina |
| `remmina-plugin-vnc` | greffon VNC pour Remmina |
| `virt-viewer` | client SPICE/VNC (`remote-viewer`) pour Proxmox |
| `deja-dup` | sauvegarde graphique GNOME (« Sauvegardes ») |
| `vlc` | lecteur multimédia |
| `gnome-tweaks` | réglages avancés du bureau GNOME |
| `smartmontools` | état de santé des disques (`smartctl`) |

### Docker et runtime NVIDIA (`05-docker.yml`)
Docker CE (`docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`,
`docker-compose-plugin`) depuis le dépôt officiel `download.docker.com`, puis
`nvidia-container-toolkit` (dépôt NVIDIA) et configuration de la runtime `nvidia` via
`nvidia-ctk runtime configure --runtime=docker`. Le pilote NVIDIA n'est pas installé.

## Bureautique
LibreOffice (avec paquets de langue française), Firefox et Thunderbird (`.deb` du dépôt APT
officiel `packages.mozilla.org`, snaps désinstallés), KeePassXC (`keepassxc-full`) et 7-Zip (`p7zip-full`).

## Pare-feu (`07-pare-feu.yml`)
`ufw` activé : entrée refusée sauf SSH (`22/tcp`), sortie autorisée. Le serveur SSH lui-même
n'est pas installé. Les ports publiés par Docker contournent `ufw`.

## zsh + Oh My Zsh + Powerlevel10k (`08-zsh.yml`)
Oh My Zsh (plugins `git`, `kubectl`, `docker`, `helm`) et Powerlevel10k, versions épinglées,
déposés dans `/etc/skel` avec un `.zshrc` et un `.p10k.zsh` (dossier `files/zsh/`), plus la
police MesloLGS NF. zsh devient le shell par défaut des **nouveaux** comptes
(`/etc/default/useradd`, `/etc/adduser.conf`) ; les comptes existants ne changent pas.

## Secrets git (`06-secrets-git.yml`)
git stocke ses identifiants dans gnome-keyring : installation de `gnome-keyring`,
`libsecret-tools` et des dépendances de compilation, compilation de `git-credential-libsecret`
(sources livrées avec git) vers `/usr/local/bin`, puis
`git config --system credential.helper`. Nécessite une session graphique pour déverrouiller
le trousseau.

## À venir
`argocd`, `cilium`, `dyff`, `gator`, `hauler`, `jf`, `kind`, `kubectl-argo-rollouts`,
`ntfy`, `opa`, `yamlfmt`.
