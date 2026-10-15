# Outils du quotidien

Statut : Implémentée (spec rédigée d'après le playbook existant)

## Objectif
Le master dispose des utilitaires en ligne de commande d'usage courant : éditeur, moniteurs
système, manipulation JSON/YAML, archives, git.

## Outils et versions
Tous en paquets APT, version du dépôt Ubuntu : `nano`, `ncdu`, `htop`, `btop`, `zsh`, `git`,
`jq`, `wget`, `unzip`, `tar`, `yq`, `ydiff`, `bat`, `musl`, `virtualbox`,
`virtualbox-guest-utils`, `virtualbox-guest-additions-iso`, `virtualbox-ext-pack`,
`gufw`, `remmina`, `remmina-plugin-rdp`, `remmina-plugin-vnc`, `virt-viewer`
(+ `apt-transport-https` et `gnupg`, déjà dans le socle `tasks/00-prerequis.yml`).

## Contraintes d'installation
- APT uniquement, non interactif, en root, sous `ansible-pull` (voir `CLAUDE.md`).
- Idempotent : 2e exécution à 0 `changed`.
- Un commentaire inline par paquet dans la liste.

## VirtualBox
- `virtualbox` : hyperviseur ; `virtualbox-guest-utils` et `virtualbox-guest-additions-iso` :
  Guest Additions et leur ISO à monter dans les VM invitées.
- `virtualbox-ext-pack` : **important** pour l'usage de VirtualBox. Il apporte USB 2.0/3.0,
  RDP (VRDP), démarrage PXE Intel, NVMe et chiffrement de disque. Sans lui, ces fonctions
  sont absentes des VM.
- Ces paquets viennent de `universe`/`multiverse` : ces composants doivent être activés.
- `virtualbox-ext-pack` télécharge l'Extension Pack d'Oracle (licence PUEL) et affiche une
  demande d'acceptation de licence debconf : en non interactif (`ansible-pull`), la réponse
  doit être préconfigurée, sinon l'installation échoue ou se bloque.

## Pare-feu, bureau à distance
- `gufw` : interface graphique pour `ufw` ; elle ne configure ni n'active le pare-feu.
- `remmina` avec les greffons RDP et VNC.
- `virt-viewer` : fournit `remote-viewer`, client SPICE pour les consoles de VM Proxmox.

## Critères d'acceptation
- `command -v nano ncdu htop btop zsh git jq wget unzip tar yq ydiff bat` trouve chaque outil
  (sous Ubuntu, `bat` peut s'appeler `batcat` : à vérifier).
- `VBoxManage list extpacks` liste l'Extension Pack.
- `command -v remote-viewer` trouve le client SPICE.
- 2e exécution du playbook : 0 `changed`.

## Hors périmètre
- Configuration de `zsh` (oh-my-zsh, `.zshrc`, shell par défaut).
- Outils Kubernetes (spec 01) et bureautique (spec 02).
