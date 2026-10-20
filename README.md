# homelab-ansible

Ansible project that provisions and manages a bare-metal Kubernetes cluster with three control-plane nodes and dedicated worker nodes. The current control-plane nodes can also run workloads when listed in the `workers` inventory group. Application deployment is handled by [Flux CD](https://fluxcd.io/), connected to a companion GitOps repository.

**Stack:** kubeadm, containerd, Flannel CNI, Flux CD, Grafana

## Infrastructure

Three Debian/Ubuntu control-plane nodes and three dedicated Debian/Ubuntu worker nodes. Node hostnames, IPs, SSH credentials, and Kubernetes network configuration are defined in `ansible/inventory.yaml`.

## Project Structure

```
ansible/
  inventory.yaml              # All hosts and variables
  playbooks/
    00-prerequisites.yaml     # System packages, containerd, kubeadm/kubelet/kubectl
    01-init-cluster.yaml      # Initialize cluster, join nodes, install Flannel
    02-join-worker.yaml      # Join workers and enable workloads on control-plane workers
    03-install-flux.yaml      # Install Flux CLI and components
    04-bootstrap-flux.yaml    # Bootstrap Flux GitOps from GitHub
    05-setup-grafana-secret.yaml  # Create Grafana admin credentials secret
    06-rolling-update.yaml    # Rolling OS updates across all nodes
    07-shutdown-cluster.yaml  # Gracefully drain and power off the cluster
    08-start-cluster.yaml     # Recover cluster after manual power-on
    09-upgrade-kubernetes.yaml # Upgrade kubeadm, kubelet, and kubectl
    10-reset-node.yaml        # Destructively remove a node from the cluster
  secrets/                    # Vault-encrypted secret files
setup.sh                      # Sets ANSIBLE_INVENTORY env var
```

## Getting Started

### Prerequisites

Ensure your SSH key is added to each node's `authorized_keys` for the SSH user defined in the inventory.

Install Ansible via pipx on your local machine and on each new node:

```bash
pipx install --include-deps ansible
```

Add entries for all cluster nodes to `/etc/hosts` on every node so hostnames used by Kubernetes resolve consistently. Use the nodes' `ansible_host` addresses from `ansible/inventory.yaml`; for the current inventory, add:

```text
192.168.194.84  zendikar.local  zendikar
192.168.194.168 dominaria.local dominaria
192.168.194.174 innistrad.local innistrad
192.168.194.238 phyrexia.local  phyrexia
192.168.194.96  kamigawa.local  kamigawa
192.168.194.157 ravnica.local  ravnica
```

### Setup

Source the environment before running any playbook:

```bash
source setup.sh
```

### Create Vault Secrets

Playbooks that reference vault-encrypted secrets require the files to exist before running. See the corresponding playbook `vars_files` entries for the expected key names.

```bash
ansible-vault create ansible/secrets/flux_secrets.yaml
ansible-vault create ansible/secrets/grafana_secrets.yaml
```

## Usage

All playbooks require `-K` (`--ask-become-pass`). Playbooks using vault secrets also require `-J` (`--ask-vault-pass`).

```bash
# Install prerequisites on all nodes
ansible-playbook ansible/playbooks/00-prerequisites.yaml -K

# Initialize the control plane and install Flannel
ansible-playbook ansible/playbooks/01-init-cluster.yaml -K

# Join all workers (including untainting control planes listed as workers)
ansible-playbook ansible/playbooks/02-join-worker.yaml -K

# Remove a specific node (destructive)
ansible-playbook ansible/playbooks/10-reset-node.yaml -e "target_node=<HOSTNAME>" -K

# Install Flux CLI and components
ansible-playbook ansible/playbooks/03-install-flux.yaml -K

# Bootstrap Flux GitOps (requires vault)
ansible-playbook ansible/playbooks/04-bootstrap-flux.yaml -K -J

# Create Grafana admin secret (requires vault)
ansible-playbook ansible/playbooks/05-setup-grafana-secret.yaml -K -J

# Rolling OS update across all nodes
ansible-playbook ansible/playbooks/06-rolling-update.yaml -K

# Gracefully drain and shut down the cluster
ansible-playbook ansible/playbooks/07-shutdown-cluster.yaml -K

# Recover cluster after manual power-on
ansible-playbook ansible/playbooks/08-start-cluster.yaml -K

# Upgrade Kubernetes through each required minor release
ansible-playbook ansible/playbooks/09-upgrade-kubernetes.yaml -K
```

### Shutdown and restart the cluster

Use the shutdown and startup playbooks as an operational pair.

```bash
# 1) Source inventory environment
source setup.sh

# 2) Gracefully drain workloads and power off nodes
ansible-playbook ansible/playbooks/07-shutdown-cluster.yaml -K

# 3) Manually power on all nodes (out-of-band)
#    Wait until SSH is reachable on every node.

# 4) Run startup recovery playbook
#    This playbook prompts interactively before continuing.
ansible-playbook ansible/playbooks/08-start-cluster.yaml -K
```

After startup completes, verify cluster health:

```bash
kubectl get nodes -o wide
kubectl get pods --all-namespaces
```

### Issues: Remove a Stale etcd Member

If a node was removed but its etcd member entry persists:

```bash
# List members
kubectl -n kube-system exec -it etcd-<HOSTNAME> -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list

# Remove by member ID
kubectl -n kube-system exec -it etcd-<HOSTNAME> -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member remove <MEMBER_ID>
```
