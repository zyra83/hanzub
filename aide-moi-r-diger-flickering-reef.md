ON ARRETE LES CONNERIES DE CLAUDE : VOIR DOC OFFICIELLE DE FIREFOX

https://support.mozilla.org/en-US/kb/install-firefox-linux#w_use-different-languages-on-firefox-deb-package

ça semble marcher mais j'avais déjà fait le lien symbolique, donc à retenter

# Plan : corriger le démarrage de Firefox (.deb PPA) bloqué par AppArmor

## Context
Sur la VM de test (Ubuntu 26.04, installée via `~/Bureau/autoinstall.yaml` puis `install-tools.yml`),
Firefox `.deb` du PPA mozillateam ne démarre pas : `Sandbox: CanCreateUserNamespace() unshare(CLONE_NEWPID): EPERM`
et `apparmor="DENIED" operation="capable" profile="firefox" capname="sys_admin"`.

Cause établie par les sorties de la VM (pas par l'autoinstall, qui ne touche pas à AppArmor) :
deux fichiers définissent un profil nommé `firefox` :
- `/etc/apparmor.d/firefox` (paquet `apparmor` d'Ubuntu) : non confinant (`flags=(unconfined)`), autorise `userns`.
- `/etc/apparmor.d/usr.bin.firefox` (paquet `firefox` du PPA, daté du 24 sept.) : profil **confinant** qui ne
  permet pas la création de user namespaces, alors que `kernel.apparmor_restrict_unprivileged_userns=1`.
C'est le second qui est chargé (sous-profils `firefox//browser_java`, `//sanitized_helper`… visibles dans `aa-status`).
Avec le snap, ce problème n'existait pas (confinement snap).

## Correction proposée (à ajouter dans `tasks/02-bureautique.yml`, après l'installation des paquets)
Utiliser le mécanisme officiel de désactivation d'un profil AppArmor, pour que le profil Ubuntu non confinant
s'applique :
1. `ansible.builtin.file` : lien symbolique `/etc/apparmor.d/disable/usr.bin.firefox` → `/etc/apparmor.d/usr.bin.firefox`
   (créer le répertoire `disable` si besoin). Survit aux mises à jour du paquet (le fichier du paquet n'est pas supprimé).
2. Si le lien vient d'être créé (`register` + `when: result.changed`) :
   - `apparmor_parser -R /etc/apparmor.d/usr.bin.firefox` (décharge le profil restrictif) ;
   - `apparmor_parser -r /etc/apparmor.d/firefox` (recharge le profil Ubuntu).
3. Mettre à jour `doc/specs/02-bureautique.md` : critère « Firefox démarre ; `aa-status` ne liste plus
   `firefox//browser_java` ; aucun refus `apparmor="DENIED"` pour firefox », et un paragraphe « Risques » : le navigateur
   n'est plus confiné par AppArmor (état identique à la configuration par défaut d'Ubuntu pour les `.deb` Firefox).
4. Mettre à jour `README.md` et `doc/a-faire.md` (point résolu / à vérifier).

Alternative (non retenue par défaut) : garder le profil du PPA et lui ajouter `userns,` dans
`/etc/apparmor.d/local/usr.bin.firefox`. Non testée, risque que d'autres refus (`capability sys_admin`) subsistent.

## À vérifier avant de coder
- Thunderbird : `dpkg -S /etc/apparmor.d/*thunderbird*` et `aa-status | grep thunderbird` sur la VM. Si le PPA fournit
  aussi `usr.bin.thunderbird` en conflit, appliquer le même schéma.

## Vérification
- Test à chaud sur la VM (avant le playbook) :
  `sudo ln -s /etc/apparmor.d/usr.bin.firefox /etc/apparmor.d/disable/ && sudo apparmor_parser -R /etc/apparmor.d/usr.bin.firefox && sudo apparmor_parser -r /etc/apparmor.d/firefox`
  puis lancer Firefox depuis la session graphique.
- Rejouer le playbook en entier sur la VM : Firefox démarre, `journalctl -k | grep 'apparmor="DENIED"' | grep firefox` est vide.
- 2e exécution du playbook : 0 `changed`.




