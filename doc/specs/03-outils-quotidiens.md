# Outils du quotidien

Statut : Implémentée (spec rédigée d'après le playbook existant)

## Objectif
Le master dispose des utilitaires en ligne de commande d'usage courant : éditeur, moniteurs
système, manipulation JSON/YAML, archives, git.

## Outils et versions
Tous en paquets APT, version du dépôt Ubuntu : `nano`, `ncdu`, `htop`, `btop`, `zsh`, `git`,
`jq`, `wget`, `unzip`, `tar`, `yq`, `ydiff`, `bat`, `musl`
(+ `apt-transport-https` et `gnupg`, déjà dans le socle `tasks/00-prerequis.yml`).

## Contraintes d'installation
- APT uniquement, non interactif, en root, sous `ansible-pull` (voir `CLAUDE.md`).
- Idempotent : 2e exécution à 0 `changed`.
- Un commentaire inline par paquet dans la liste.

## Critères d'acceptation
- `command -v nano ncdu htop btop zsh git jq wget unzip tar yq ydiff bat` trouve chaque outil
  (sous Ubuntu, `bat` peut s'appeler `batcat` : à vérifier).
- 2e exécution du playbook : 0 `changed`.

## Hors périmètre
- Configuration de `zsh` (oh-my-zsh, `.zshrc`, shell par défaut).
- Outils Kubernetes (spec 01) et bureautique (spec 02).
