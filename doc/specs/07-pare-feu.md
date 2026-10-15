# Pare-feu (ufw)

Statut : Implémentée (non testée en réel : à valider en VM)

## Objectif
Le master filtre le trafic réseau : tout le trafic entrant est bloqué sauf SSH (22/tcp) ;
tout le trafic sortant est autorisé. Le pare-feu est actif dès la fin du premier démarrage.

## Outils et versions
- `ufw` : version du dépôt Ubuntu (déjà tiré par `gufw`, installé explicitement).

## Politique
| Sens     | Politique | Exception           |
|----------|-----------|---------------------|
| Entrant  | refusé    | `22/tcp` (SSH)      |
| Sortant  | autorisé  | —                   |
| Routé    | inchangé (refusé par défaut) | — |

## Source d'installation
- Paquet APT `ufw`, pilotage par la commande `ufw` (pas de collection Ansible : le playbook
  doit rester autonome sous `ansible-pull`).
- Ordre impératif : politiques par défaut → règle SSH → activation (`ufw --force enable`),
  pour ne jamais activer le pare-feu sans la règle SSH.

## Contraintes d'installation
- Non interactif, en root, sous `ansible-pull` (voir `CLAUDE.md`) : `--force` évite la
  confirmation d'activation.
- Idempotent : les commandes ne s'exécutent que si l'état lu (`ufw status verbose`) diffère ;
  2e exécution à 0 `changed`.

## Critères d'acceptation
- `ufw status verbose` affiche `Status: active`, `Default: deny (incoming), allow (outgoing)`
  et une règle `22/tcp ALLOW IN Anywhere`.
- Depuis une autre machine : le port 22 répond, un autre port ouvert sur le master est filtré.
- 2e exécution du playbook : 0 `changed`.

## Hors périmètre
- Installation et durcissement du serveur SSH (`openssh-server`) : cette spec ouvre le port,
  elle n'installe pas le service.
- Limitation de débit des connexions SSH (`ufw limit`), restriction par réseau source.
- Ports publiés par Docker : Docker écrit ses propres règles iptables et contourne `ufw`.
  Un conteneur avec `-p` reste joignable malgré la politique entrante.
- Test dans un conteneur : `ufw` nécessite `iptables` et les privilèges noyau ; valider en VM.
