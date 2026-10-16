# zsh, Oh My Zsh et Powerlevel10k

Statut : Implémentée (config zsh testée dans un HOME factice ; playbook non testé en réel : à valider en VM)

## Objectif
Tout nouvel utilisateur créé sur le master reçoit, via `/etc/skel`, un environnement zsh prêt à
l'emploi : Oh My Zsh avec le thème Powerlevel10k, sans configuration manuelle ni assistant
interactif. zsh devient le shell par défaut des nouveaux comptes.

## Outils et versions
| Élément        | Version                     | Source                                              |
|----------------|-----------------------------|-----------------------------------------------------|
| `zsh`          | dépôt Ubuntu (déjà en spec 03) | APT                                              |
| Oh My Zsh      | commit `964ddf7` (`ohmyzsh_version`) | `https://github.com/ohmyzsh/ohmyzsh` (git clone) |
| Powerlevel10k  | `v1.20.0` (`powerlevel10k_version`) | `https://github.com/romkatv/powerlevel10k` (git clone) |
| Police MesloLGS NF | commit `145eb9f` (`meslolgs_fonts_version`) | `https://github.com/romkatv/powerlevel10k-media` (4 `.ttf`) |

## Source d'installation
- Pas de `install.sh` d'Oh My Zsh (il lance `chsh` et peut démarrer un zsh interactif) :
  `ansible.builtin.git` vers `/etc/skel/.oh-my-zsh`, version épinglée (clone superficiel :
  `depth: 1` avec `refspec` égal au commit, sinon Ansible ignore `depth` pour un SHA).
- Powerlevel10k cloné dans `/etc/skel/.oh-my-zsh/custom/themes/powerlevel10k`.
- Fichiers de configuration versionnés **dans ce dépôt** (le playbook doit rester autonome,
  voir `CLAUDE.md`) et copiés par `ansible.builtin.copy` :
  - `files/zsh/zshrc` → `/etc/skel/.zshrc` (`ZSH_THEME="powerlevel10k/powerlevel10k"`,
    `plugins=(git kubectl docker helm)`, chargement de `~/.p10k.zsh`) ;
  - `files/zsh/p10k.zsh` → `/etc/skel/.p10k.zsh` (style « lean » fourni par Powerlevel10k
    `v1.20.0`, `config/p10k-lean.zsh`, pour éviter l'assistant au premier lancement ; chacun
    peut le refaire avec `p10k configure`).
- Police : installée dans `/usr/local/share/fonts/` (puis `fc-cache -f`), indispensable à
  l'affichage des icônes de Powerlevel10k dans le terminal.
- Shell par défaut des nouveaux comptes : `SHELL=/bin/zsh` dans `/etc/default/useradd` et
  `DSHELL=/bin/zsh` dans `/etc/adduser.conf`.

## Contraintes d'installation
- Non interactif, en root, sous `ansible-pull` (voir `CLAUDE.md`) ; réseau tout juste monté :
  clones GitHub avec nouvelles tentatives (`retries`/`until`).
- Idempotent : 2e exécution à 0 `changed` (`git` avec `version` épinglée, `copy` sans
  écrasement inutile, `lineinfile` pour les fichiers de configuration shell).
- Droits : `/etc/skel` appartient à root ; les dépôts clonés doivent être lisibles par tous
  (`0755`/`0644`), car ils sont copiés tels quels dans chaque `$HOME`.
- Tag `minimal` : la liste est incluse dans l'installation minimale (`zsh` en fait partie).
- Nouvelle liste de tâches `tasks/08-zsh.yml` importée dans `install-tools.yml` ; les versions
  vont dans `vars:` de `install-tools.yml`.

## Critères d'acceptation
- `/etc/skel/.oh-my-zsh`, `/etc/skel/.zshrc` et `/etc/skel/.p10k.zsh` existent.
- `sudo adduser --disabled-password --gecos "" essai` crée un compte dont le shell est
  `/usr/bin/zsh` ; `sudo -u essai -i zsh -ic 'echo $ZSH_THEME'` renvoie
  `powerlevel10k/powerlevel10k` sans invite de configuration.
- `fc-list | grep -i meslolgs` liste la police.
- 2e exécution du playbook : 0 `changed`.

## Hors périmètre
- Comptes existants : `/etc/skel` n'est copié qu'à la création d'un compte. Le compte créé par
  `autoinstall.yml` est un compte de secours : on ne lui ajoute rien (décision actée).
- Changer le shell de `root`.
- Choix de la police dans le terminal : réglage par utilisateur, hors playbook.
- Mises à jour d'Oh My Zsh / Powerlevel10k après installation (la version est épinglée).

## Décisions
- Thème : Powerlevel10k retenu.
- Plugins Oh My Zsh activés dans `files/zsh/zshrc` : `git`, `kubectl`, `docker`, `helm`.

## Alternative à considérer : Starship
Powerlevel10k est en mode maintenance (correctifs seulement, plus d'évolutions). Si le thème
devient un problème, [Starship](https://starship.rs) est l'alternative suggérée :
prompt multi-shell écrit en Rust, activement maintenu, configuré par un seul fichier
`~/.config/starship.toml` (donc `/etc/skel/.config/starship.toml`), sans dépendre d'Oh My Zsh.
Il demande lui aussi une police Nerd Font. Non retenu pour l'instant : le prompt Powerlevel10k
est plus rapide à l'affichage grâce à son mode « instant prompt », et la configuration est
déjà connue.

## Écarté pour le moment
Deux plugins externes, non livrés avec Oh My Zsh (à cloner dans
`/etc/skel/.oh-my-zsh/custom/plugins/`, version épinglée, puis à ajouter à `plugins=(…)`) :
- `zsh-autosuggestions` : affiche en gris la suite probable de la commande d'après l'historique
  (`→` pour accepter).
- `zsh-syntax-highlighting` : colore la ligne en direct (commande valide en vert, inconnue en
  rouge) pour repérer les fautes avant de valider.

Écartés pour l'instant : seuls `git`, `kubectl`, `docker` et `helm` sont retenus. À reconsidérer
plus tard, sans impact sur le reste de la spec.

## Questions ouvertes (à trancher avant d'implémenter)
Aucune.
