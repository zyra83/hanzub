# Outils Kubernetes

Statut : Implémentée (exemple de spec, reflète le playbook actuel ; les points « À corriger » ne le sont pas encore)

## Objectif
Le master Ubuntu dispose, dès la fin du premier démarrage, des outils en ligne de commande
pour administrer et tester Kubernetes : `kubectl`, `helm`, `k9s`, `minikube`.

## Outils et versions
| Outil      | Version                  | Source d'installation                                              |
|------------|--------------------------|--------------------------------------------------------------------|
| `kubectl`  | branche `v1.34` (var `kubernetes_minor_version`) | dépôt APT `pkgs.k8s.io/core:/stable:/v1.34/deb/` |
| `helm`     | dernière du dépôt        | dépôt APT `packages.buildkite.com/helm-linux/helm-debian`          |
| `k9s`      | `v0.50.9` (var `k9s_version`)     | release GitHub `derailed/k9s`, archive `k9s_Linux_<arch>.tar.gz` |
| `minikube` | `v1.37.0` (var `minikube_version`) | `storage.googleapis.com/minikube/releases/<version>/minikube-linux-<arch>` |

## Contraintes d'installation
- Cibles : Ubuntu/Debian uniquement (échec explicite sinon), architectures `amd64` et `arm64`
  (échec explicite sinon), via la variable `binary_architecture`.
- Prérequis APT : `apt-transport-https`, `ca-certificates`, `curl`, `gnupg`.
- Clés des dépôts APT dans `/etc/apt/keyrings/` (`.asc`), référencées par `signed-by=` ;
  pas de `apt-key`.
- `k9s` et `minikube` installés dans `/usr/local/bin`, mode `0755`.
- Exécution en root, non interactive, sous `ansible-pull` (voir `CLAUDE.md`).

## Critères d'acceptation
- `kubectl version --client` affiche `v1.34.x`.
- `helm version` s'exécute sans erreur.
- `k9s version` affiche `v0.50.9`.
- `minikube version` affiche `v1.37.0`.
- 2e exécution du playbook : 0 `changed`.
- Sur une machine non Debian ou d'architecture non supportée : le playbook s'arrête
  dans les `pre_tasks` avec un message clair.

## À corriger (écarts avec le playbook actuel)
- `minikube` utilise `force: true` : il est retéléchargé à chaque exécution, donc la 2e
  exécution n'est pas à 0 `changed`. Cible : ne retélécharger que si la version change
  (par exemple `dest` versionné ou vérification de `minikube version`).
- Les variables `kubernetes_keyring` et `helm_keyring` (`.gpg`) sont inutilisées : supprimer
  ou utiliser au lieu des chemins `.asc` en dur.
- Dépôt Kubernetes ajouté avec `lineinfile`, Helm avec `apt_repository` : harmoniser.

## Hors périmètre
- Création d'un cluster (`minikube start`, `kind`) et configuration de `kubeconfig`.
- Outils du quotidien (`jq`, `bat`, `zsh`…) : voir une spec dédiée.
- Autres outils Kubernetes de la liste TODO (`argocd`, `cilium`, `kind`…) : specs séparées.
