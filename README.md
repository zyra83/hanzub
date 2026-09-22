# hanzub
Installer Ansible, puis lancer le playbook sur la machine locale :

```bash
sudo apt update && sudo apt install -y ansible
ansible-playbook -i localhost, -c local -K install-tools.yml
```

Le playbook installe `nano`, `btop`, `zsh`, `kubectl`, `helm`, `k9s` et `minikube` sur Ubuntu/Debian.
