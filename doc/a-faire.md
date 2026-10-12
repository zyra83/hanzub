# À relire / à faire (session du 2026-10-12)

Modifications non commitées, testées sur `provisioning@192.168.137.166` (Ubuntu 26.04 « resolute »).

## À relire (modifs faites)
- `tasks/02-bureautique.yml` :
  - suppression de toute la détection du bureau (stat des sessions, `set_fact`, `debug`, `block`/`when`) ;
    les tâches sont désormais au premier niveau. Reste l'assert « système Ubuntu ».
  - PPA Mozilla : plus de `ppa:` ni de `gpg --keyserver` (échec « Unable to get required signing key »
    sur 26.04). Clé téléchargée en HTTPS dans `/etc/apt/keyrings/mozillateam.asc`
    (empreinte `738BEB9321D1AAEC13EA9391AEBDF4819BE21867`), puis source `deb [signed-by=…]`
    avec `{{ ansible_distribution_release }}`. `software-properties-common` n'est plus installé.
- `doc/specs/02-bureautique.md` : cible « Ubuntu Desktop uniquement », étape 0 (garde Ubuntu seule)
  et étape 1 mises à jour.
- `README.md` : section « Bureautique » sans mention « poste avec bureau uniquement ».
- `CLAUDE.md` : exception « PPA Mozilla géré par `apt_repository` » retirée.

## À faire
1. **`ydiff` introuvable en APT** (tâche « Installer divers outils du travail quotidien »,
   `tasks/03-outils-quotidiens.yml`) : le run s'arrête là, le reste de la spec 03 n'est pas
   installé. Options : `pipx` (à ajouter en prérequis) ou retrait de la liste. À trancher.
2. Corriger le tableau de `doc/specs/02-bureautique.md` : mentionne encore `ppa:mozillateam/ppa`.
3. Relancer le playbook en entier sur la VM, puis une 2e fois pour vérifier 0 `changed`.
4. Vérifier les critères d'acceptation de la spec 02 (versions Firefox/Thunderbird, langue FR,
   `snap list firefox thunderbird` vide, priorité 1001).
5. Pas de `--syntax-check` ni de `--check` complet avant poussée : le `--check` échoue sur
   l'assert PPA car `apt-cache policy` est ignoré en mode check (attendu).
6. Commit puis push (tout ce qui est poussé est appliqué par `ansible-pull`).

## Pour relancer à distance (mot de passe SSH)
- Nécessite `sshpass` en local.
- La cible utilise `sudo-rs` (invite traduite, bloque `become` d'Ansible) :
  ajouter `-e ansible_become_exe=sudo.ws`.
- Exemple (mots de passe dans un fichier d'extra-vars hors dépôt) :
  `ansible-playbook -i 192.168.137.166, -u provisioning -c ssh -e @creds.yml -e ansible_become_exe=sudo.ws install-tools.yml`
- `sudo-rs` risque aussi de gêner `ansible-pull` en production : à vérifier (root, donc a priori sans `become`).
