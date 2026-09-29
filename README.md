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

## Bureautique (poste avec bureau uniquement)
LibreOffice (avec paquets de langue française), Firefox et Thunderbird (`.deb` du PPA
`mozillateam`, snaps désinstallés), KeePassXC et 7-Zip (`p7zip-full`). Ignorée sur un système
sans environnement graphique.

## À venir
`argocd`, `cilium`, `dyff`, `gator`, `hauler`, `jf`, `kind`, `kubectl-argo-rollouts`,
`ntfy`, `opa`, `yamlfmt`.
