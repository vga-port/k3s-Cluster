# homelab-k3s-ansible

An opinionated, batteries-included homelab Kubernetes bootstrapper built on k3s.

This repo:
- Installs a k3s control-plane on hosts in the `k3s_masters` group
- Optionally installs Longhorn for storage
- Installs MetalLB + Traefik ingress
- Deploys a configurable set of homelab apps via Helm

## Quickstart

```bash
git clone <your-repo-url> homelab-k3s-ansible
cd homelab-k3s-ansible

# Edit inventory
nano inventory/homelab/hosts.ini
nano inventory/homelab/group_vars/*.yml

# Dry-run (syntax check)
ansible-playbook -i inventory/homelab/hosts.ini cluster.yml --syntax-check

# Run
ansible-playbook -i inventory/homelab/hosts.ini cluster.yml
```

See `inventory/homelab/group_vars/` for configuration and `values/` for app Helm values.
