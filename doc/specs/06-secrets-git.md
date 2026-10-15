# Secrets git (gnome-keyring)

Statut : Implémentée (non testée en réel : à valider en VM)

## Objectif
git stocke les identifiants HTTPS dans le trousseau gnome-keyring (Secret Service) via
`git-credential-libsecret`, pour tous les utilisateurs de la machine.

## Outils et versions
- `build-essential`, `pkg-config`, `libsecret-1-0`, `libsecret-1-dev`, `libglib2.0-dev` :
  dépendances de compilation (version du dépôt Ubuntu).
- `libsecret-tools` : commande `secret` (diagnostic du trousseau).
- `gnome-keyring` : trousseau de secrets.
- `git-credential-libsecret` : compilé depuis les sources livrées avec git.

## Source d'installation
- Paquets APT.
- Ubuntu ne livre pas le binaire : seules les sources sont dans
  `/usr/share/doc/git/contrib/credential/libsecret/`. Elles sont copiées dans un répertoire
  temporaire, compilées avec `make`, et le binaire est installé dans
  `/usr/local/bin/git-credential-libsecret` (mode `0755`).
- Configuration : `git config --system credential.helper /usr/local/bin/git-credential-libsecret`
  (`/etc/gitconfig`, car le playbook tourne en root et ne connaît pas l'utilisateur final).

## Contraintes d'installation
- Non interactif, en root, sous `ansible-pull` (voir `CLAUDE.md`).
- Idempotent : compilation ignorée si le binaire existe ; 2e exécution à 0 `changed`.

## Critères d'acceptation
- `/usr/local/bin/git-credential-libsecret` existe et est exécutable.
- `git config --system credential.helper` renvoie son chemin.
- `secret search --all` s'exécute dans une session graphique.
- 2e exécution du playbook : 0 `changed`.

## Hors périmètre
- Déverrouillage du trousseau et saisie des jetons : nécessitent une session graphique.
- Recompilation du helper lors d'une mise à jour de git ou de libsecret.
- `credential.helper cache` (essai abandonné).

## Référence externe

Voir le tuto https://www.baeldung.com/linux/gnome-keyring-git-credentials-provider#bd-overview
