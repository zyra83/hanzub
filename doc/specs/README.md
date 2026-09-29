# Spécifications

Une spec = un fichier = un sujet, numéroté pour l'ordre d'implémentation
(ex. `01-outils-kubernetes.md`, `02-shell-zsh.md`). Ces fichiers font foi pour le playbook.

## Gabarit

```markdown
# <Titre>

Statut : À faire | En cours | Implémentée

## Objectif
Ce que la machine doit avoir/faire à l'issue du premier démarrage.

## Outils et versions
- outil : version (épinglée ou « dernière »)

## Source d'installation
APT / release GitHub / autre, URL si connue.

## Critères d'acceptation
- commande de vérification attendue (ex. `k9s version` renvoie v0.50.9)
- 2e exécution du playbook : 0 `changed`

## Hors périmètre
Ce qu'il ne faut pas faire.
```
