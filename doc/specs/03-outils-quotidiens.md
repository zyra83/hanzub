# Outils du quotidien

Statut : Implémentée (spec rédigée d'après le playbook existant)

## Objectif
Le master dispose des utilitaires en ligne de commande d'usage courant : éditeur, moniteurs
système, manipulation JSON/YAML, archives, git.

## Outils et versions
Tous en paquets APT, version du dépôt Ubuntu : `nano`, `ncdu`, `htop`, `btop`, `zsh`, `git`,
`jq`, `wget`, `unzip`, `tar`, `yq`, `ydiff`, `bat`, `musl`, `virtualbox`,
`virtualbox-guest-additions-iso`, `virtualbox-ext-pack`,
`gufw`, `remmina`, `remmina-plugin-rdp`, `remmina-plugin-vnc`, `virt-viewer`, `deja-dup`,
`vlc`, `gnome-tweaks`, `smartmontools`
(+ `apt-transport-https` et `gnupg`, déjà dans le socle `tasks/00-prerequis.yml`).

## Installation minimale (tag `minimal`)
Seuls `nano`, `zsh`, `git`, `wget`, `unzip`, `tar`, `bat`, `gufw`, `deja-dup`, `gnome-tweaks`
et `smartmontools` sont installés avec `--tags minimal` (première tâche de
`tasks/03-outils-quotidiens.yml`). Les autres paquets forment une seconde tâche, sans tag.

## Contraintes d'installation
- APT uniquement, non interactif, en root, sous `ansible-pull` (voir `CLAUDE.md`).
- Idempotent : 2e exécution à 0 `changed`.
- Un commentaire inline par paquet dans la liste.

## VirtualBox
- `virtualbox` : hyperviseur ; `virtualbox-guest-additions-iso` : ISO des Guest Additions à
  monter dans les VM invitées. `virtualbox-guest-utils` n'est pas installé (inutile sur l'hôte).
- `virtualbox-ext-pack` : **important** pour l'usage de VirtualBox. Il apporte USB 2.0/3.0,
  RDP (VRDP), démarrage PXE Intel, NVMe et chiffrement de disque. Sans lui, ces fonctions
  sont absentes des VM.
- Ces paquets viennent de `universe`/`multiverse` : ces composants doivent être activés.
- `virtualbox-ext-pack` télécharge l'Extension Pack d'Oracle (licence PUEL) et affiche une
  demande d'acceptation de licence debconf : en non interactif (`ansible-pull`), la réponse
  doit être préconfigurée, sinon l'installation échoue ou se bloque.

## Pare-feu, bureau à distance
- `gufw` : interface graphique pour `ufw` ; elle ne configure ni n'active le pare-feu
  (voir `doc/specs/07-pare-feu.md`).

## Sauvegarde, disques, multimédia
- `deja-dup` : sauvegarde graphique GNOME, chiffrée. Seul le paquet est installé : la cible
  (disque, NAS, cloud) dépend de l'utilisateur et se règle dans l'application.
- `smartmontools` : `smartctl` pour l'état de santé des disques.
- `vlc` : lecteur multimédia ; `gnome-tweaks` : réglages avancés du bureau GNOME.
- `remmina` avec les greffons RDP et VNC.
- `virt-viewer` : fournit `remote-viewer`, client SPICE pour les consoles de VM Proxmox.

## Critères d'acceptation
- `command -v nano ncdu htop btop zsh git jq wget unzip tar yq ydiff bat` trouve chaque outil
  (sous Ubuntu, `bat` peut s'appeler `batcat` : à vérifier).
- `VBoxManage list extpacks` liste l'Extension Pack.
- `command -v remote-viewer deja-dup vlc gnome-tweaks smartctl` trouve chaque outil.
- `command -v remote-viewer` trouve le client SPICE.
- 2e exécution du playbook : 0 `changed`.

## Hors périmètre
- Configuration de `zsh` (oh-my-zsh, `.zshrc`, shell par défaut) : voir `doc/specs/08-zsh.md`.
- Configuration de la cible de sauvegarde de `deja-dup`.
- Outils Kubernetes (spec 01) et bureautique (spec 02).
