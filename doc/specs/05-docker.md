# Docker et runtime NVIDIA

Statut : Implémentée (non testée sur une machine Ubuntu : à valider en VM)

## Objectif
Le master dispose de Docker CE et de la runtime NVIDIA pour Docker, afin d'exécuter des
conteneurs avec accès aux GPU NVIDIA.

## Outils et versions
| Outil                    | Version  | Paquets APT                                                                   |
|--------------------------|----------|-------------------------------------------------------------------------------|
| Docker CE                | dernière | `docker-ce`, `docker-ce-cli`, `containerd.io`, `docker-buildx-plugin`, `docker-compose-plugin` |
| NVIDIA Container Toolkit | dernière | `nvidia-container-toolkit`                                                    |

## Source d'installation
- Docker : dépôt officiel `https://download.docker.com/linux/<distribution>`, clé dans
  `/etc/apt/keyrings/docker.asc`, suite = codename de la distribution. Remplace `docker.io`.
- NVIDIA : `https://nvidia.github.io/libnvidia-container/stable/deb/<arch>`, clé dans
  `/etc/apt/keyrings/nvidia-container-toolkit.asc`.
- Runtime : `nvidia-ctk runtime configure --runtime=docker` (écrit `/etc/docker/daemon.json`),
  exécuté seulement si `nvidia` n'y figure pas encore, puis redémarrage de Docker.

## Contraintes d'installation
- Non interactif, en root, sous `ansible-pull` (voir `CLAUDE.md`).
- Idempotent : 2e exécution à 0 `changed`.

## Critères d'acceptation
- `docker --version`, `docker compose version` et `docker buildx version` s'exécutent.
- `systemctl is-active docker` renvoie `active`.
- `docker info | grep -i nvidia` montre la runtime `nvidia`.
- Avec le pilote NVIDIA : `docker run --rm --gpus all ubuntu nvidia-smi` affiche le GPU.
- 2e exécution du playbook : 0 `changed`.

## Hors périmètre
- Installation du pilote NVIDIA (prérequis séparé, absent sur une machine sans GPU).
- Ajout d'utilisateurs au groupe `docker` (le playbook ne connaît pas l'utilisateur final).
- Configuration de Docker au-delà de la runtime NVIDIA.
