# homelab-ansible

Ansible project that provisions and manages a 3-node bare-metal Kubernetes cluster. All nodes serve as both control plane and workers. Application deployment is handled by [Flux CD](https://fluxcd.io/), connected to a companion GitOps repository.

**Stack:** kubeadm, containerd, Flannel CNI, Flux CD, Grafana

## Infrastructure

3 Debian/Ubuntu nodes, each serving as control plane + worker. Node hostnames, IPs, SSH credentials, and Kubernetes network configuration are defined in `ansible/inventory.yaml`.

## Project Structure

```
ansible/
  inventory.yaml              # All hosts and variables
  playbooks/
    00-prerequisites.yaml     # System packages, containerd, kubeadm/kubelet/kubectl
    01-init-cluster.yaml      # Initialize cluster, join nodes, install Flannel
    02-reset-node.yaml        # Destructively remove a node from the cluster
    03-install-flux.yaml      # Install Flux CLI and components
    04-bootstrap-flux.yaml    # Bootstrap Flux GitOps from GitHub
    05-setup-grafana-secret.yaml  # Create Grafana admin credentials secret
    06-rolling-update.yaml    # Rolling OS updates across all nodes
  secrets/                    # Vault-encrypted secret files
setup.sh                      # Sets ANSIBLE_INVENTORY env var
```

## Getting Started

### Prerequisites

Ensure your SSH key is added to each node's `authorized_keys` for the SSH user defined in the inventory.

Install Ansible on your local machine:

```bash
pipx install --include-deps ansible
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

# Initialize the Kubernetes cluster
ansible-playbook ansible/playbooks/01-init-cluster.yaml -K

# Remove a specific node (destructive)
ansible-playbook ansible/playbooks/02-reset-node.yaml -e "target_node=<HOSTNAME>" -K

# Install Flux CLI and components
ansible-playbook ansible/playbooks/03-install-flux.yaml -K

# Bootstrap Flux GitOps (requires vault)
ansible-playbook ansible/playbooks/04-bootstrap-flux.yaml -K -J

# Create Grafana admin secret (requires vault)
ansible-playbook ansible/playbooks/05-setup-grafana-secret.yaml -K -J

# Rolling OS update across all nodes
ansible-playbook ansible/playbooks/06-rolling-update.yaml -K
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
