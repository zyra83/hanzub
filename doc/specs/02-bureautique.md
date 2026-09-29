# Outils de bureautique et navigateur

Statut : Implémentée (non testée sur une machine Ubuntu : à valider en VM)

## Objectif
Le master Ubuntu Desktop dispose, dès la fin du premier démarrage, d'une suite bureautique,
d'un client mail et d'un navigateur, en français. Thunderbird et Firefox sont installés en
`.deb` depuis le PPA Mozilla Team, et non en snap.

## Outils et versions
| Outil         | Version                       | Source / paquets APT                                              |
|---------------|-------------------------------|-------------------------------------------------------------------|
| LibreOffice   | celle du dépôt Ubuntu         | `libreoffice`, `libreoffice-l10n-fr`, `libreoffice-help-fr`       |
| Thunderbird   | dernière du PPA               | `ppa:mozillateam/ppa` : `thunderbird`, `thunderbird-locale-fr`    |
| Firefox       | dernière du PPA               | `ppa:mozillateam/ppa` : `firefox`, `firefox-locale-fr`            |
| KeePassXC     | celle du dépôt Ubuntu         | `keepassxc`                                                       |
| 7-Zip complet | celle du dépôt Ubuntu         | `p7zip-full` (interprété comme « 7zfull » ; sur Ubuntu récent, `7zip` existe aussi : à confirmer) |

## Contraintes d'installation
- APT uniquement une fois les snaps retirés : pas de snap ni de Flatpak pour ces outils.
- Cible : Ubuntu Desktop (environnement graphique présent). Sur un système sans bureau,
  ces tâches ne doivent pas être exécutées.
- Non interactif, en root, sous `ansible-pull` (voir `CLAUDE.md`).
- Idempotent : 2e exécution à 0 `changed`.

## Ordre des opérations (important)
Ubuntu installe Firefox et Thunderbird en snap d'office. Dans `tasks/02-bureautique.yml` :
0. **Garde** : n'exécuter que sur un poste avec bureau (`/usr/share/wayland-sessions` ou
   `/usr/share/xsessions` présent) et sous Ubuntu ; sinon, ignorer (bureau absent) ou échouer (pas Ubuntu).
1. **Ajouter le PPA** `ppa:mozillateam/ppa` (`ansible.builtin.apt_repository`).
2. **Poser une préférence APT** `/etc/apt/preferences.d/mozilla-ppa` :
   `Package: firefox* thunderbird*`, `Pin: release o=LP-PPA-mozillateam`, `Pin-Priority: 1001`.
   Sans elle, le paquet de transition d'Ubuntu (qui réinstalle le snap) l'emporte.
3. **Autoriser les mises à jour automatiques** depuis le PPA (`LP-PPA-mozillateam:${distro_codename}`
   dans `Unattended-Upgrade::Allowed-Origins`, fichier dédié dans `/etc/apt/apt.conf.d/`).
4. **Vérifier** via `apt-cache policy` que le PPA fournit bien `firefox` et `thunderbird` ;
   sinon échec explicite, **sans avoir touché aux snaps**.
5. **Désinstaller les snaps** `firefox` et `thunderbird` (`snap remove --purge`), seulement
   s'ils sont présents (`snap list <nom>`).
6. **Installer les paquets** `.deb` (`state: latest`, `allow_downgrade: true`, car le paquet de
   transition d'Ubuntu peut déjà être installé et doit être remplacé par la version du PPA).

Le PPA est ajouté par `apt_repository` (qui gère sa propre clé) : exception à la règle des clés
`.asc` dans `/etc/apt/keyrings/`.

## Risques et précautions
- `snap remove --purge` **supprime les données** du snap (profils Firefox/Thunderbird,
  marque-pages, mails locaux). Acceptable sur un master fraîchement installé ; le playbook ne
  doit donc **jamais** retirer un snap dont le `.deb` PPA n'est pas déjà la source active
  (sinon, un utilisateur qui aurait réinstallé le snap perdrait son profil à chaque exécution).
  Condition : ne supprimer que si le snap est présent **et** que le PPA est vérifié (étapes 1 à 4).
  Conséquence : un snap réinstallé volontairement est de nouveau supprimé à chaque exécution.
- Dépôt tiers ajouté au premier démarrage : si le PPA est indisponible, le navigateur manque.
  Prévoir un échec explicite plutôt qu'un retour silencieux au snap.
- Le PPA doit proposer la version Ubuntu du master (`distro_codename`) ; à vérifier avant
  une montée de version d'Ubuntu.

## Critères d'acceptation
- `libreoffice --version` s'exécute et LibreOffice s'ouvre en français.
- `dpkg -s libreoffice-l10n-fr` indique `Status: install ok installed`.
- `snap list firefox thunderbird` ne renvoie aucun des deux.
- `apt-cache policy firefox` et `apt-cache policy thunderbird` montrent le PPA
  `mozillateam` comme source installée, avec la priorité 1001.
- `firefox --version` et `thunderbird --version` s'exécutent ; les interfaces sont en français.
- `keepassxc-cli --version` s'exécute ; `7z` est disponible (`command -v 7z`).
- `apt install --dry-run firefox` ne réinstalle pas le snap.
- 2e exécution du playbook : 0 `changed`.

## Hors périmètre
- Autres navigateurs (Chromium, etc.).
- Outils créatifs (Gimp, Inkscape…) et lecteurs PDF : specs séparées si besoin.
- Configuration des comptes mail, des profils, des extensions et des politiques Firefox.
- Sauvegarde ou migration des données des snaps supprimés.
