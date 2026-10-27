# FAQ — Problèmes courants sous Ubuntu

Problèmes fréquents rencontrés sous Ubuntu/Debian, avec leur cause probable et la solution.

## Sommaire

- [APT et paquets](#apt-et-paquets)
- [Réseau](#réseau)
- [Disque et système de fichiers](#disque-et-système-de-fichiers)
- [Démarrage et services](#démarrage-et-services)
- [Affichage et session graphique](#affichage-et-session-graphique)
- [Droits et sudo](#droits-et-sudo)
- [VirtualBox](#virtualbox)
- [Ansible / ce dépôt](#ansible--ce-dépôt)
- [Commandes de diagnostic utiles](#commandes-de-diagnostic-utiles)
- [Logiciels courants par profil](#logiciels-courants-par-profil)

---

## APT et paquets

### `Could not get lock /var/lib/dpkg/lock-frontend`

**Cause** : un autre processus APT tourne (mises à jour automatiques, `unattended-upgrades`, Logiciels).

```bash
sudo lsof /var/lib/dpkg/lock-frontend      # identifier le processus
sudo systemctl status unattended-upgrades
```

Attendre la fin du processus. Ne supprimer le verrou qu'en dernier recours, quand aucun processus APT ne tourne :

```bash
sudo rm /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/cache/apt/archives/lock
sudo dpkg --configure -a
```

### `dpkg was interrupted, you must manually run 'dpkg --configure -a'`

```bash
sudo dpkg --configure -a
sudo apt -f install
```

### `Unable to locate package xxx`

**Causes** : index APT périmé, dépôt `universe` désactivé, ou paquet absent de la version d'Ubuntu.

```bash
sudo apt update
sudo add-apt-repository universe
apt-cache policy xxx
```

### `NO_PUBKEY` / `The repository ... is not signed`

**Cause** : clé GPG du dépôt manquante ou mal référencée.

Utiliser une clé dans `/etc/apt/keyrings/` et la référencer avec `signed-by=` dans la source
(`apt-key` est obsolète) :

```bash
sudo install -d -m 0755 /etc/apt/keyrings
sudo curl -fsSL <URL_DE_LA_CLE> -o /etc/apt/keyrings/nom.asc
# dans la source : deb [signed-by=/etc/apt/keyrings/nom.asc] <URL> <suite> main
```

### `404 Not Found` sur un dépôt

**Cause** : dépôt retiré ou déplacé (ex. : ancienne URL Helm `baltocdn`), ou version d'Ubuntu en fin de vie.

- Vérifier l'URL dans `/etc/apt/sources.list.d/`.
- Version EOL : passer à une version supportée, ou utiliser `old-releases.ubuntu.com` en dernier recours.

### `Hash Sum mismatch` / `File has unexpected size`

**Cause** : miroir en cours de synchronisation ou cache corrompu.

```bash
sudo rm -rf /var/lib/apt/lists/*
sudo apt clean
sudo apt update
```

### `E: Unmet dependencies` / paquets cassés

```bash
sudo apt --fix-broken install
sudo apt update && sudo apt full-upgrade
```

### Les paquets « Snap » sont lents ou posent problème

```bash
snap list
sudo snap refresh
sudo snap remove --purge <nom>
```

Préférer le paquet APT ou le binaire officiel quand l'application a des soucis d'accès aux fichiers (confinement).

---

## Réseau

### Pas de connexion après le démarrage

```bash
ip a                        # interface et adresse IP
nmcli device status         # état NetworkManager
sudo systemctl restart NetworkManager
```

### Résolution DNS en échec (`Temporary failure in name resolution`)

**Cause** : `systemd-resolved` mal configuré ou DNS injoignable.

```bash
resolvectl status
sudo systemctl restart systemd-resolved
ping 1.1.1.1                # si OK, le problème vient du DNS
```

### Téléchargements GitHub / GCS en échec juste après le boot

**Cause** : le réseau n'est pas encore opérationnel.

Le service doit dépendre de `network-online.target` (`After=` et `Wants=`), et les tâches de téléchargement
doivent utiliser `retries` / `delay`.

---

## Disque et système de fichiers

### `No space left on device`

```bash
df -h                       # espace par partition
df -i                       # inodes saturés ?
sudo du -xh / --max-depth=1 | sort -h | tail
sudo apt autoremove --purge && sudo apt clean
sudo journalctl --vacuum-size=200M
```

Vérifier aussi les anciennes révisions snap et les images Docker :

```bash
snap list --all | awk '/disabled/{print $1, $3}'
docker system prune
```

### `No space left on device` alors que le disque physique a de la place (LVM)

**Cause** : avec l'installation LVM par défaut d'Ubuntu, le volume logique (`ubuntu-lv`) n'occupe pas
forcément tout le groupe de volumes (souvent environ la moitié). L'espace libre existe mais n'est pas alloué.

```bash
sudo vgs                    # colonne VFree : espace libre dans le groupe de volumes
sudo lvs                    # taille actuelle du volume logique
df -h /

# étendre le volume logique à tout l'espace libre, puis redimensionner le système de fichiers
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

L'option `-r` redimensionne le système de fichiers en même temps (ext4/XFS, à chaud, sans démontage).
Sans elle, lancer ensuite `sudo resize2fs /dev/ubuntu-vg/ubuntu-lv` (ext4) ou `sudo xfs_growfs /` (XFS).

Si `VFree` est nul mais que le disque a été agrandi (VM, disque virtuel) : agrandir d'abord la partition, puis le volume physique.

```bash
sudo growpart /dev/sda 3    # numéro de la partition LVM (cloud-guest-utils)
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE -r /dev/ubuntu-vg/ubuntu-lv
```

### Partition en lecture seule

**Cause** : erreur disque détectée, le système remonte la partition en `ro`.

```bash
dmesg | grep -i "error\|read-only"
sudo mount -o remount,rw /
sudo fsck -f /dev/sdXN      # sur une partition démontée
```

### Partition Windows (NTFS) impossible à monter

**Cause** : Windows hibernait ou « démarrage rapide » activé.

Désactiver le démarrage rapide dans Windows, ou :

```bash
sudo ntfsfix /dev/sdXN
```

---

## Démarrage et services

### Un service échoue au démarrage

```bash
systemctl --failed
systemctl status <service>
journalctl -u <service> -b --no-pager
```

### Démarrage très lent

```bash
systemd-analyze
systemd-analyze blame | head
systemd-analyze critical-chain
```

Cause fréquente : `systemd-networkd-wait-online` ou `NetworkManager-wait-online` en attente d'une interface sans lien.

### Retrouver les journaux du démarrage précédent

```bash
journalctl -b -1
journalctl -p err -b
```

---

## Affichage et session graphique

### Écran noir ou boucle de connexion après une mise à jour

**Cause** : pilote graphique (souvent NVIDIA) incompatible avec le noyau.

```bash
# depuis un TTY (Ctrl+Alt+F3)
sudo ubuntu-drivers devices
sudo ubuntu-drivers autoinstall
sudo apt install --reinstall ubuntu-desktop
```

Démarrer sur un noyau précédent depuis GRUB (« Advanced options ») pour dépanner.

### Mauvaise résolution ou écran externe non détecté

```bash
xrandr                      # session X11
```

Sous Wayland, utiliser Paramètres → Écrans. Essayer aussi de basculer X11/Wayland depuis l'écran de connexion (roue crantée).

### Partage d'écran ou capture impossible (Wayland)

**Cause** : certaines applications ne supportent pas Wayland.

Choisir la session « Ubuntu sur Xorg » à la connexion, ou installer `xdg-desktop-portal-gnome`.

### Polices ou caractères manquants

```bash
sudo apt install fonts-noto fonts-liberation ttf-mscorefonts-installer
fc-cache -fv
```

---

## Droits et sudo

### `user is not in the sudoers file`

Depuis un compte administrateur ou en mode recovery :

```bash
sudo usermod -aG sudo <utilisateur>
```

Se déconnecter puis se reconnecter pour appliquer le changement de groupe.

### `Permission denied` sur Docker (`/var/run/docker.sock`)

```bash
sudo usermod -aG docker $USER
newgrp docker               # ou se reconnecter
```

### `sudo: unable to resolve host xxx`

**Cause** : le nom d'hôte n'est pas dans `/etc/hosts`.

```bash
hostname
# ajouter dans /etc/hosts :
127.0.1.1   <hostname>
```

### Mot de passe oublié

Contacter l'administrateur. Les comptes sont centralisés dans l'AD (Active Directory), et la réinitialisation
d'un mot de passe, y compris celui d'un compte local, relève des administrateurs.

---

## VirtualBox

### `The kernel driver is not installed (rc=-1908)` / module `vboxdrv` absent

**Cause** : modules non compilés pour le noyau courant, ou Secure Boot actif.

```bash
sudo apt install linux-headers-$(uname -r) dkms
sudo /sbin/vboxconfig
mokutil --sb-state          # Secure Boot ?
```

Avec Secure Boot, signer les modules via une clé MOK (Machine Owner Key) ; ne pas désactiver Secure Boot.

### Utilisateur sans accès aux périphériques USB

```bash
sudo usermod -aG vboxusers $USER
```

Se reconnecter ensuite.

### Licence de l'Extension Pack demandée à l'installation

Elle est pré-acceptée via `debconf` dans ce dépôt. Pour le faire à la main :

```bash
echo virtualbox-ext-pack virtualbox-ext-pack/license select true | sudo debconf-set-selections
```

### Presse-papier ou résolution invités non fonctionnels

Installer les additions invité dans la VM (ou `virtualbox-guest-utils` si disponible pour l'invité), puis redémarrer la VM.

---

## Ansible / ce dépôt

### Le playbook échoue au premier démarrage

Le playbook est lancé par `ansible-pull` via un service systemd one-shot. Il est idempotent : le relancer suffit.

```bash
systemctl status <service-ansible-pull>
journalctl -u <service-ansible-pull> -b --no-pager
```

### `Syntax Error while loading YAML` ou erreur de syntaxe

```bash
ansible-playbook -i localhost, -c local --syntax-check install-tools.yml
ansible-lint install-tools.yml
```

### Une tâche APT échoue juste après l'ajout d'un dépôt

**Cause** : le cache n'a pas été mis à jour. Ajouter `update_cache: true` à la tâche d'installation suivante.

### Un binaire est retéléchargé à chaque exécution (`changed` à chaque run)

**Cause** : `force: true` sur la tâche de téléchargement (ex. minikube). Le playbook doit afficher 0 `changed`
au second passage.

### Simuler avant d'appliquer

```bash
ansible-playbook -i localhost, -c local --check --diff -K install-tools.yml
```

### Installer seulement le socle minimal

```bash
ansible-playbook -i localhost, -c local -K install-tools.yml --tags minimal
```

---

## Commandes de diagnostic utiles

| Besoin | Commande |
|---|---|
| Version d'Ubuntu | `lsb_release -a` |
| Noyau | `uname -r` |
| Matériel | `lspci`, `lsusb`, `lscpu`, `free -h` |
| Services en échec | `systemctl --failed` |
| Journal d'erreurs | `journalctl -p err -b` |
| Occupation disque | `df -h`, `ncdu /` |
| Processus gourmands | `top`, `htop` |
| Ports en écoute | `ss -tulpn` |
| Paquet propriétaire d'un fichier | `dpkg -S /chemin/fichier` |
| Version candidate d'un paquet | `apt-cache policy <paquet>` |

---

## Logiciels courants par profil

Légende : ● usage courant · ○ usage occasionnel · (vide) peu ou pas utilisé.
Colonne **SILL** : ✔ = logiciel présent (à ma connaissance) dans le
[Socle interministériel de logiciels libres](https://code.gouv.fr/sill/) ; « — » = non libre ou absent.
Cette colonne est indicative, établie de mémoire et non vérifiée en ligne : à confirmer sur code.gouv.fr/sill.

| Logiciel | Catégorie | Dev | DevOps | Bureautique | SILL |
|---|---|:-:|:-:|:-:|:-:|
| **Bureautique et collaboration** | | | | | |
| LibreOffice | Suite bureautique | ○ | ○ | ● | ✔ |
| Thunderbird | Messagerie | ○ | ○ | ● | ✔ |
| Tchap | Messagerie instantanée | ○ | ○ | ○ | ✔ |
| Mattermost (client Desktop) | Messagerie d'équipe | ○ | ● | ○ | ✔ |
| Microsoft Teams | Messagerie d'équipe | ● | ● | ● | — |
| [Visio](https://visio.numerique.gouv.fr/) | Visioconférence | ○ | ○ | ○ | — |
| Zotero | Gestion bibliographique | | | ○ | ✔ |
| Notion / Obsidian | Prise de notes | ○ | ○ | ○ | — |
| **Internet et sécurité du poste** | | | | | |
| Firefox | Navigateur | ● | ● | ● | ✔ |
| Chromium / Google Chrome | Navigateur | ● | ● | ● | — |
| KeePassXC | Gestionnaire de mots de passe | ● | ● | ● | ✔ |
| Bitwarden | Gestionnaire de mots de passe | ○ | ● | ○ | ✔ |
| 7-Zip / p7zip | Archives | ○ | ○ | ○ | ✔ |
| **Médias et graphisme** | | | | | |
| VLC | Lecteur multimédia | ○ | ○ | ● | ✔ |
| GIMP | Retouche d'image | ○ | | ○ | ✔ |
| Inkscape | Dessin vectoriel | ○ | | ○ | ✔ |
| Krita | Dessin numérique | | | ○ | ✔ |
| Blender | 3D | ○ | | | ✔ |
| OBS Studio | Capture / streaming | ○ | ○ | ○ | ✔ |
| Audacity | Édition audio | | | ○ | ✔ |
| Kdenlive | Montage vidéo | | | ○ | ✔ |
| Draw.io (diagrams.net) | Diagrammes | ● | ● | ○ | ✔ |
| Figma | Maquettes UI | ○ | | ○ | — |
| **Édition de code** | | | | | |
| Visual Studio Code | Éditeur / IDE | ● | ● | | — |
| JetBrains (IntelliJ, PyCharm, GoLand…) | IDE | ● | ○ | | — |
| Vim / Neovim | Éditeur terminal | ● | ● | | ✔ |
| Cursor / Zed | Éditeur orienté IA / performance | ○ | ○ | | — |
| **Langages et runtimes** | | | | | |
| Git | Gestion de versions | ● | ● | | ✔ |
| Python | Langage | ● | ● | ○ | ✔ |
| Node.js | Runtime JavaScript | ● | ○ | | ✔ |
| OpenJDK | Runtime Java | ● | ○ | | ✔ |
| Go | Langage | ○ | ● | | ✔ |
| Rust (rustup / cargo) | Langage | ○ | ○ | | ✔ |
| **Données et API** | | | | | |
| DBeaver | Client SQL | ● | ○ | | ✔ |
| Postman | Client d'API | ● | ○ | | — |
| Bruno / Insomnia | Client d'API | ○ | ○ | | — |
| curl / jq / yq | Outils CLI | ● | ● | | ✔ |
| **Conteneurs et Kubernetes** | | | | | |
| Docker (Engine / Desktop) | Conteneurs | ● | ● | | ✔ |
| Podman | Conteneurs sans démon | ○ | ● | | ✔ |
| kubectl | Client Kubernetes | ○ | ● | | ✔ |
| Helm | Gestionnaire de charts | ○ | ● | | ✔ |
| k9s | Interface terminal Kubernetes | | ● | | ✔ |
| Minikube / k3s | Cluster local | ○ | ● | | ✔ |
| Kind | Cluster local dans Docker | ○ | ● | | ✔ |
| Lens / Headlamp | Interface Kubernetes | | ○ | | — |
| argocd (CLI) | Client Argo CD (GitOps) | | ● | | ✔ |
| cilium (CLI) | Client Cilium (réseau Kubernetes) | | ○ | | ✔ |
| **Infrastructure as Code et CI/CD** | | | | | |
| Ansible | Configuration | ○ | ● | | ✔ |
| Terraform | IaC | ○ | ● | | — |
| OpenTofu | IaC | | ○ | | ✔ |
| vault / bao (CLI) | Client Vault / OpenBao (secrets) | | ● | | ✔ |
| **Réseau et sécurité** | | | | | |
| Wireshark | Analyse réseau | ○ | ● | | ✔ |
| nmap | Scan réseau | | ● | | ✔ |
| OpenSSH | Accès distant | ● | ● | | ✔ |
| WireGuard / OpenVPN | VPN | ○ | ● | ○ | ✔ |
| Trivy | Scan de vulnérabilités | ○ | ● | | ✔ |
| **Shell et productivité terminal** | | | | | |
| Zsh + Oh My Zsh / Powerlevel10k | Shell | ● | ● | | ✔ |
| tmux | Multiplexeur de terminal | ○ | ● | | ✔ |
| fzf / ripgrep / bat | Outils CLI modernes | ● | ● | | ✔ |
| **Virtualisation** | | | | | |
| VirtualBox | Machines virtuelles | ● | ● | ○ | ✔ |
| KVM / QEMU / libvirt | Machines virtuelles | ○ | ● | | ✔ |
| Vagrant | VM reproductibles | ○ | ○ | | — |

Remarques :
- Les « vrais » logiciels à la mode non présents au SILL (Teams, Notion, Cursor, Figma, Postman, Terraform…)
  sont signalés par « — » : propriétaires ou sous licence non libre (Terraform est passé en BSL, d'où OpenTofu).
- Les profils se recoupent : un développeur utilise aussi la bureautique, et un profil DevOps reprend la plupart des outils d'un développeur.
