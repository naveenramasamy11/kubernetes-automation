# Kubernetes Automation with Ansible

> Fully automated Kubernetes cluster provisioning using Ansible — supports both Debian and RedHat based systems.

---

## Overview

This project automates the end-to-end setup of a Kubernetes cluster using Ansible roles. It handles everything from OS-level prerequisites (swap, SELinux, kernel settings, package installation) through control plane initialization and worker node joining — all idempotent and re-runnable.

**Supported OS families:** Ubuntu / Debian · CentOS / RHEL
**Kubernetes tooling:** `kubeadm`, `kubelet`, `kubectl`, `kubernetes-cni`
**CNI Plugin:** [Calico](https://docs.tigera.io/calico/latest/about/)

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                   Ansible Control Node               │
│                  (runs k8s.yaml playbook)            │
└────────────┬────────────────────────┬────────────────┘
             │                        │
             ▼                        ▼
  ┌──────────────────┐     ┌──────────────────────┐
  │   Master Node    │     │    Worker Nodes       │
  │  kube-master-1   │     │  kube-worker-1 / 2   │
  │                  │     │                      │
  │  kubeadm init    │────▶│  kubeadm join        │
  │  Calico CNI      │     │                      │
  └──────────────────┘     └──────────────────────┘
```

**Playbook execution order:**

```
Play 1 → k8s-prereq      (all nodes)   — OS prep, packages, Docker
Play 2 → k8s             (master only) — kubeadm init, Calico CNI
Play 3 → k8s-node-joining (all)        — worker nodes join cluster
```

---

## Repository Structure

```
kubernetes-automation/
├── k8s.yaml                        # Main playbook
├── ansible.cfg                     # Ansible configuration
├── inventory.yaml                  # Inventory (hosts + groups)
│
├── k8s-prereq/                     # Role: OS prerequisites
│   ├── tasks/main.yml
│   ├── defaults/main.yml           # Overridable variables
│   ├── handlers/main.yml
│   └── meta/main.yml
│
├── k8s/                            # Role: Kubernetes control plane init
│   ├── tasks/main.yml
│   ├── defaults/main.yml           # pod_network_cidr, calico_version
│   └── meta/main.yml
│
└── k8s-node-joining/               # Role: Worker node join
    ├── tasks/main.yml
    └── meta/main.yml
```

---

## Prerequisites

On the **Ansible control node**:

- Ansible 2.10+
- SSH access to all target nodes
- Python 3 on all target nodes

On **target nodes**:

- Ubuntu 20.04+ or CentOS 7/8 / RHEL 7/8
- Minimum 2 CPUs, 2 GB RAM (Kubernetes requirement)
- Nodes reachable by hostname or IP

---

## Inventory Setup

Copy and adapt the inventory template below. Group your nodes under `masters` and `workers`.

```yaml
# inventory.yaml
all:
  children:
    masters:
      hosts:
        kube-master-1:
          ansible_host: 192.168.0.106
      vars:
        ansible_ssh_user: ubuntu
        ansible_become: yes
        ansible_python_interpreter: python3

    workers:
      hosts:
        kube-worker-1:
          ansible_host: 192.168.0.110
        kube-worker-2:
          ansible_host: 192.168.0.111
      vars:
        ansible_ssh_user: ubuntu
        ansible_become: yes
        ansible_python_interpreter: python3
```

---

## Configuration

Key variables are defined in each role's `defaults/main.yml` and can be overridden via `group_vars`, `host_vars`, or `-e` on the command line.

| Variable | Default | Description |
|---|---|---|
| `pod_network_cidr` | `10.244.0.0/16` | Pod network CIDR passed to `kubeadm init`. Must not overlap with host network. |
| `calico_version` | `v3.27.3` | Calico CNI version to install. |
| `necessary_packages` | `kubelet, kubeadm, kubectl, kubernetes-cni` | Kubernetes packages installed on all nodes. |
| `services` | `docker, kubelet` | Services started and enabled after install. |

To override a variable without editing the role:

```bash
ansible-playbook -i inventory.yaml k8s.yaml -e "pod_network_cidr=10.244.0.0/16"
```

---

## Usage

### Full cluster provisioning

```bash
ansible-playbook -i inventory.yaml k8s.yaml
```

### Run a specific phase using tags

```bash
# Only run prerequisites
ansible-playbook -i inventory.yaml k8s.yaml --tags prereq

# Only initialize the master
ansible-playbook -i inventory.yaml k8s.yaml --tags masters

# Only join worker nodes
ansible-playbook -i inventory.yaml k8s.yaml --tags join
```

### Dry run (check mode)

```bash
ansible-playbook -i inventory.yaml k8s.yaml --check
```

### Verbose output

```bash
ansible-playbook -i inventory.yaml k8s.yaml -v
```

---

## Role Details

### `k8s-prereq`
Prepares all nodes for Kubernetes:
- Adds Kubernetes package repositories (YUM for RedHat, APT for Debian)
- Installs Docker and Kubernetes packages (`kubelet`, `kubeadm`, `kubectl`)
- Disables swap permanently (runtime + `/etc/fstab`)
- Configures `net.bridge.bridge-nf-call-iptables` sysctl and loads `br_netfilter`
- Disables SELinux / firewalld on RedHat systems
- Populates `/etc/hosts` with all inventory node entries

### `k8s`
Initializes the Kubernetes control plane on the first master node:
- Runs `kubeadm init` with configurable `pod_network_cidr` (idempotent — skipped if cluster is already initialized)
- Sets up `~/.kube/config` for `kubectl` access
- Deploys the Calico CNI plugin at a pinned version

### `k8s-node-joining`
Joins all worker nodes to the cluster:
- Generates a fresh join token from the master
- Checks each worker for an existing `/etc/kubernetes/kubelet.conf` before joining (idempotent)
- Join token is never printed to stdout (`no_log: true`)

---

## Security Notes

- SSH host key checking is **enabled** — ensure your `known_hosts` is populated before running, or pre-accept host keys.
- The kubeadm join token is handled with `no_log: true` throughout and is never logged to Ansible output.
- Ansible Vault is recommended for managing `ansible_ssh_user` credentials in production. See the [Ansible Vault docs](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

---

## Contributing

Bug reports and pull requests are welcome.

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-fix`)
3. Commit your changes
4. Open a Pull Request

---

## Author

**Naveen Ramasamy** — Infrastructure Automation Engineer
[github.com/naveenramasamy11](https://github.com/naveenramasamy11)