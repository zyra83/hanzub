# hanzub

Playbook Ansible (`install-tools.yml`, qui importe les listes de tâches de `tasks/`) qui installe les outils de dev
Kubernetes et utilitaires quotidiens sur Ubuntu/Debian (APT + binaires GitHub/GCS).
Cible : la machine locale. Langue du projet : français (noms de tâches, commentaires, README).

## Mode d'exécution en production
Le playbook est lancé par `ansible-pull` (depuis ce dépôt git), via un service systemd
one-shot, au **premier démarrage** du master Ubuntu fraîchement installé avec un
`autoinstall.yml`. Le service et l'`autoinstall.yml` ne sont pas dans ce dépôt.

Conséquences pour le playbook :
- Il s'exécute en **root**, sans TTY ni `-K` : ne jamais dépendre d'un mot de passe sudo
  ni d'une saisie interactive.
- Il est exécuté à un moment où le réseau vient tout juste de monter : les téléchargements
  (dépôts APT, GitHub, GCS) doivent rester tolérants (le service doit attendre
  `network-online.target`).
- Il doit rester **autonome** : tout fichier ou variable nécessaire doit être dans le dépôt
  (pas de `--extra-vars` ni d'inventaire externe). `ansible-pull` utilise `localhost` en
  connexion locale, d'où `hosts: all`.
- Le fichier doit être à la racine du dépôt et sera appelé explicitement
  (`ansible-pull … install-tools.yml`), car le nom par défaut d'`ansible-pull` est `local.yml`.
- Il doit être idempotent : un échec au premier boot doit pouvoir être relancé sans casse.
- Tout ce qui est poussé sur la branche suivie par `ansible-pull` est appliqué sur les
  machines : tester avant de pousser.

## Specs
Les spécifications sont dans `doc/specs/` (gabarit dans `doc/specs/README.md`) et font foi.
- Les lire avant de coder ; implémenter uniquement ce qu'elles demandent.
- En cas de conflit avec le playbook existant ou d'ambiguïté, le signaler plutôt que deviner.
- Une fois codée, passer le statut de la spec à « Implémentée » et mettre à jour le README.

## Commandes
- Exécution : `ansible-playbook -i localhost, -c local -K install-tools.yml`
- Syntaxe : `ansible-playbook -i localhost, -c local --syntax-check install-tools.yml`
- Simulation : ajouter `--check --diff -K`
- Lint : `ansible-lint install-tools.yml` (si installé)

## Structure du playbook (règle à respecter à chaque édition)
`install-tools.yml` est un **orchestrateur** : un seul play (vars, pre_tasks = asserts APT +
architecture) qui appelle des listes de tâches via `ansible.builtin.import_tasks`.
**Ne jamais ajouter de tâche directement dans `install-tools.yml`.**

- Une spec `doc/specs/NN-nom.md` ⇔ une liste de tâches `tasks/NN-nom.yml` (même numéro,
  même nom), importée dans `install-tools.yml` dans l'ordre des numéros. En tête du fichier
  de tâches : `# Spec : doc/specs/NN-nom.md`.
- Seule exception : `tasks/00-prerequis.yml` (socle commun : prérequis APT, répertoire des clés),
  sans spec.
- Une spec « À faire » n'est pas importée tant qu'elle n'est pas implémentée ; à l'implémentation,
  créer le fichier de tâches, l'importer et passer la spec à « Implémentée ».
- Les variables (versions, `binary_architecture`) restent dans `vars:` de `install-tools.yml` ;
  les fichiers de tâches ne contiennent que des tâches (pas d'en-tête de play).
- Chemins d'import relatifs au playbook (`tasks/…`) : compatible avec `ansible-pull`, qui exécute
  le playbook depuis la copie du dépôt. Pas de sous-playbooks (`import_playbook`) : ils
  répéteraient hôtes, `become` et variables.
- Le nom d'une tâche importée doit rester unique et lisible dans la sortie d'`ansible-pull`
  (pour le suivi) ; le nom du bloc d'import indique la spec concernée.

## Fichiers
- `README.md` : usage ; à tenir à jour quand la liste d'outils change
- `tasks/` : listes de tâches, une par spec
- `doc/specs/` : les specs (voir plus haut)
- Le bas du playbook contient une liste TODO d'outils à ajouter en commentaire
  (argocd, cilium, dyff, gator, hauler, jf, kind, kubectl-argo-rollouts, ntfy, opa, yamlfmt…)

## Conventions
- Modules en FQCN (`ansible.builtin.*`), noms de tâches en français, verbe à l'infinitif.
- Versions épinglées dans `vars:` (`k9s_version`, `minikube_version`, `kubernetes_minor_version`) ;
  ne jamais coder une version en dur dans une tâche.
- Architecture via `binary_architecture` (amd64/arm64) pour les binaires téléchargés.
- Binaires installés dans `/usr/local/bin` avec mode `0755`.
- Clés APT dans `/etc/apt/keyrings/*.asc` + `signed-by=` dans la source (pas de `apt-key`, pas de dearmor). Exception : le PPA Mozilla, géré par `apt_repository`.
- Le playbook doit rester idempotent : relancé deux fois, 0 `changed` la 2e fois
  (attention à `force: true` sur minikube qui retélécharge à chaque run).
- Commentaire inline pour chaque paquet de `tasks/03-outils-quotidiens.yml`.

## Pièges connus
- Les vars `kubernetes_keyring` et `helm_keyring` (chemins `.gpg`) sont inutilisées :
  les tâches utilisent des chemins `.asc` en dur. Unifier plutôt que d'ajouter un 3e chemin.
- `apt-transport-https` et `gnupg` sont installés à la fois dans `00-prerequis` et
  `03-outils-quotidiens` (inoffensif, à dédoublonner).
- La refactorisation en `tasks/` n'a pas pu être validée avec `--syntax-check` (ansible absent
  de la machine de rédaction) : lancer la vérification avant de pousser.
- Le dépôt Kubernetes est ajouté avec `lineinfile` alors que Helm utilise `apt_repository`.
- Le dépôt Helm vient de packages.buildkite.com (l'ancienne URL baltocdn ne marche plus).
- Après ajout d'un dépôt APT, `update_cache: true` est nécessaire à l'installation suivante.

## Workflow
- Tester avec `--check` puis en réel sur une VM/conteneur Ubuntu avant de commiter.
- Commits courts en français ou anglais, dans le style de l'historique.
- Mettre à jour le README (section « Organisation » et « Ce qui est installé ») quand une spec
  est implémentée ou un outil ajouté.
