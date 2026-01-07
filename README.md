# K3s Homelab Cluster (Ansible) - MetalLB + Traefik + Longhorn + Apps + Rancher


This repo deploys a multi-node k3s cluster using Ansible, then installs common homelab components using Helm and kubectl through a staged set of roles.

It targets home LAN setups (no cloud load balancers).

Currently only tested on:

  ✅ Debian (tested on version 11)
  
  ✅ Ubuntu (tested on version 22.04)


## What it deploys


Core cluster:
ets home LAN setups (no cloud load balancers).

What it deploys

Core cluster:
- k3s controller (master)
- k3s workers joining the cluster
- optional control-plane tainting/labels

Platform services:

- MetalLB (L2 LoadBalancer IPs)
- Traefik (Ingress)
- Longhorn (storage)

Apps (configurable):

- PostgreSQL (Bitnami)
- Redis (Bitnami)
- Nextcloud
- Immich (+ required library PVC manifest)
- kube-prometheus-stack
- Grafana
- cert-manager
- Rancher

Tooling:

- k9s (installed on controller)

## Playbook flow (what runs and, where)

The `cluster.yml` runs in this order:

1. cluster_bootstrap (hosts: `k3s_masters`)
Bootstraps k3s controller

2. cluster_workers (hosts: `k3s_workers`)
Joins worker nodes to cluster

3. cluster_policy (hosts: `k3s_masters`)
Applies taints/labels (e.g. make control-plane unschedulable)

4. helm_bootstrap (hosts: `k3s_masters`)
Installs helm + kubectl tooling needed by later roles

5. cluster_network + cluster_storage (hosts: `k3s_masters`)
Installs MetalLB/Traefik and Longhorn and configures required objects

6. cluster_apps (hosts: `k3s_masters`)
Installs your configured apps from Helm values + manifests

7. cluster_monitoring (hosts: `k3s_masters`)
Optional monitoring stack

8. cluster_rancher (hosts: `k3s_masters`)
Optional cert-manager/Rancher installation

9. tools_k9s (hosts: `k3s_masters`)
Installs k9s for debugging

## Requirements (don’t skip this)
### Ansible controller (the machine you run the playbook from)

- Ansible installed
- SSH key-based access to all nodes
- kubernetes.core collection installed

Example (Ubuntu/Debian):

``` bash
sudo apt update
sudo apt install -y ansible python3-pip
ansible-galaxy collection install kubernetes.core
```

### Target nodes (controller + workers)

Each node needs:

- Linux (Currently only tested on Ubuntu/ Debian distros)
- SSH reachable from Ansible controller
- Sudo access for ansible_user
- Working outbound internet + DNS (charts + images must download)
- Time sync working (NTP)


# Quick start
### 1) Clone
```bash
git clone <YOUR_REPO_URL>
cd <YOUR_REPO_DIR>
```

### 2) Configure inventory

Edit `inventory/hosts.ini`:

```ini
[k3s_masters]
ctrlr ansible_user=ctrlr-test ansible_host=192.168.1.115

[k3s_workers]
wrkr ansible_user=wrkr-test ansible_host=192.168.1.116
wrkr1 ansible_user=wrkr1-test ansible_host=192.168.1.117

[k3s_cluster:children]
k3s_masters
k3s_workers
```

Test connectivity (ensure that you are in the `k3s-Clsuter` directory):

```bash
ansible -i inventory/homelab/hosts.ini k3s_cluster -m ping
```
### 3) Configure variables

Configure your main vars in `group_vars/all.yml` (or your repo’s var file).

Things you must set correctly:

- `k3s_server_ip`
- MetalLB address pool range (must not overlap)
- Which apps are enabled
- Ensure `values/*.yaml` files exist for enabled apps

### 4) Run install
```bash
ansible-playbook -i inventory/hosts.ini cluster.yml -K
```

# Values + manifests
### Helm values

For each enabled app with a `values_file`, you must have a matching file in:

```bash
values/<name>.yaml
```

The playbooks copy those values onto the controller node and install charts using them.

### Immich library PVC manifest

Immich requires a pre-created PVC (library storage). This repo expects:

```bash
manifests/immich-library-pvc.yaml
```

Example PVC (Longhorn):

```bash
apiVersion: v1
kind: Namespace
metadata:
  name: immich
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: immich-library
  namespace: immich
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 100Gi
```

# Post-install: using kubectl and k9s

On the controller node:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
sudo kubectl get nodes -o wide
```

k9s should be installed by the `tools_k9s` role. If you install manually:

```bash
cd /tmp
wget -q https://github.com/derailed/k9s/releases/latest/download/k9s_linux_amd64.deb
sudo apt install -y ./k9s_linux_amd64.deb
rm -f ./k9s_linux_amd64.deb
k9s
```

# Common failure modes (and what they actually mean)

1) MetalLB webhook errors on first run

If you see errors like “no endpoints available for service metallb-webhook-service”:

- The CR is being applied before the webhook pod is ready.

- Fix is not “rerun and pray”; fix is rollout wait + retry in cluster_network.

2) MetalLB IPAddressPool overlap

If you see:

`CIDR ... overlaps with already defined CIDR`

That means you already created a pool. Delete it:

```bash
kubectl -n metallb-system get ipaddresspools.metallb.io
kubectl -n metallb-system delete ipaddresspool <name>
```

3) Longhorn “volume not ready for workloads”

PVC bound ≠ volume ready. Longhorn can bind PVCs and still refuse attach if Longhorn components aren’t healthy.

Check:

```bash
kubectl -n longhorn-system get pods
kubectl -n longhorn-system get nodes.longhorn.io
kubectl -n longhorn-system describe pod -l app=longhorn-manager
```

4) Helm chart version “not found”

If you pin versions and the chart repo no longer contains them, Helm fails.
Fix: update version pins or stop pinning for a homelab.

5) Docker/registry DNS timeouts

If you get `lookup registry-1.docker.io: i/o timeout`, it’s your DNS/network.
Rerunning isn’t a fix; it’s a coin toss.

# Wiping the cluster (“nuke”)

On each node:

```bash
sudo /usr/local/bin/k3s-uninstall.sh || true
sudo /usr/local/bin/k3s-agent-uninstall.sh || true

sudo rm -rf /etc/rancher /var/lib/rancher /var/lib/kubelet /var/lib/cni /etc/cni /run/flannel
sudo reboot
```

Then re-run the playbook.

# Contributors / debugging issues

If you raise an issue, include:

```bash
ansible-playbook -i inventory/hosts.ini cluster.yml -K -vvv
```

And:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get events -A --sort-by=.metadata.creationTimestamp | tail -n 50
```
Redact any secrets.

# Thank you!

This repo would not of been possible without the use of the following:

- k3s-io/k3s-ansible
- timothystewart6/k3s-ansible 
- geerlingguy/turing-pi-cluster
- 212850a/k3s-ansible
