# AGENTS.md

## Project Overview

Ansible project that provisions and manages a 3-node bare-metal Kubernetes cluster on a homelab. All three nodes serve as both control plane and workers (taints removed). Application deployment is handled by Flux CD, connected to a companion GitOps repository.

**Stack:** kubeadm, containerd, Flannel CNI, Flux CD, Grafana

## Project Structure

```
ansible/
  inventory.yaml          # Single inventory file with all hosts and variables
  playbooks/              # Numbered playbooks (00-06), executed sequentially
    00-prerequisites.yaml   # System packages, containerd, kubeadm/kubelet/kubectl
    01-init-cluster.yaml    # Initialize cluster, join nodes, install Flannel
    02-reset-node.yaml      # Destructively remove a node from the cluster
    03-install-flux.yaml    # Install Flux CLI and components
    04-bootstrap-flux.yaml  # Bootstrap Flux GitOps from GitHub
    05-setup-grafana-secret.yaml  # Create Grafana admin credentials secret
    06-rolling-update.yaml  # Rolling OS updates across all nodes
  secrets/                # Vault-encrypted secret files (never commit plaintext)
setup.sh                  # Sets ANSIBLE_INVENTORY env var
```

There are no roles, `ansible.cfg`, `group_vars/`, or `host_vars/` directories. All variables are defined inline in `ansible/inventory.yaml` under `all.vars`.

## Infrastructure

3 Debian/Ubuntu (apt-based) nodes, each serving as control plane + worker. Python is managed via Miniconda at `/opt/miniconda3/envs/ansible/`.

**All host-specific values** -- node hostnames, LAN IPs, Kubernetes network IPs, SSH user and key, pod/service CIDRs, and Flux GitOps target -- are defined in `ansible/inventory.yaml`. Always consult that file for current infrastructure details.

## Conventions

### Commit Messages

Follow Conventional Commits: `type(ansible): description`

- **Types:** `feat`, `fix`, `refactor`, `docs`, `chore`
- Sentence-case description, no trailing period
- Examples: `feat(ansible): Add shutdown playbook`, `fix(ansible): Correct CNI plugin path`

### Ansible Style

- Use **FQCNs** for all modules: `ansible.builtin.apt`, `ansible.builtin.command`, `kubernetes.core.k8s`, etc.
- Use `true`/`false` for booleans, not `yes`/`no`
- Always set `changed_when` and `failed_when` on `command`, `shell`, and `raw` tasks
- Use `.yaml` file extension, not `.yml`
- Name new playbooks with a numbered prefix: `XX-description.yaml`
- Task names should be descriptive and sentence-case

## Security

- **Never** commit plaintext secrets, vault passwords, SSH keys, or kubeconfigs
- Vault-encrypted files live in `ansible/secrets/`
- Vault password is provided at runtime with `-J` (`--ask-vault-pass`)
- Refer to `.gitignore` for the full list of excluded sensitive patterns

## Running Playbooks

1. Source the environment: `source setup.sh` (or `export ANSIBLE_INVENTORY=./ansible/inventory.yaml`)
2. All playbooks require `-K` (`--ask-become-pass`) for privilege escalation
3. Playbooks that use vault secrets also require `-J` (`--ask-vault-pass`)

```bash
# Example: run prerequisites
ansible-playbook ansible/playbooks/00-prerequisites.yaml -K

# Example: bootstrap flux (needs vault)
ansible-playbook ansible/playbooks/04-bootstrap-flux.yaml -K -J

# Example: reset a specific node
ansible-playbook ansible/playbooks/02-reset-node.yaml -e "target_node=<HOSTNAME>" -K
```

## Warnings

- **`02-reset-node.yaml`** is destructive -- it fully drains, removes, and wipes a node from the cluster
- **`06-rolling-update.yaml`** performs rolling reboots -- nodes are drained and updated one at a time, never simultaneously
- Always validate cluster health after any changes (`kubectl get nodes`, etcd health checks)
