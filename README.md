# Vagrant K3s Cluster

This repository provides a local three-node K3s lab cluster using Vagrant and Ubuntu 24.04.

## Topology

| Node | Role | IP | SSH |
| --- | --- | --- | --- |
| `k3s-master` | K3s server / control plane | `192.168.56.33` | `vagrant ssh k3s-master` |
| `k3s-worker-1` | K3s agent / worker | `192.168.56.34` | `vagrant ssh k3s-worker-1` |
| `k3s-worker-2` | K3s agent / worker | `192.168.56.35` | `vagrant ssh k3s-worker-2` |

Default VM credentials:

```text
username: vagrant
password: vagrant
```

## Prerequisites

- Vagrant
- A supported Vagrant provider, such as VMware Desktop or VirtualBox
- Ansible
- `sshpass` if using password-based SSH from the included Ansible inventory

On macOS:

```bash
brew install ansible sshpass
```

Optional Vagrant plugin:

```bash
vagrant plugin install vagrant-disksize
```

## VM Lifecycle

Start all VMs:

```bash
vagrant up
```

Check VM status:

```bash
vagrant status
```

SSH into a node:

```bash
vagrant ssh k3s-master
vagrant ssh k3s-worker-1
vagrant ssh k3s-worker-2
```

Stop all VMs:

```bash
vagrant halt
```

Restart all VMs:

```bash
vagrant reload
```

Destroy all VMs:

```bash
vagrant destroy -f
```

## Install The Cluster

The `Vagrantfile` only creates and provisions the base Ubuntu VMs. K3s and cluster add-ons are installed with Ansible.

Recommended install order:

```bash
vagrant up

cd ansible
ansible-playbook install-k3s.yml
ansible-playbook install-dependencies.yml
ansible-playbook install-argocd.yml
```

For a clean K3s reinstall on existing VMs:

```bash
cd ansible
ansible-playbook install-k3s.yml -e k3s_reset_cluster=true
```

## Ansible Playbooks

| Playbook | Purpose |
| --- | --- |
| `ansible/install-k3s.yml` | Installs the K3s server and workers, creates default namespaces, and writes kubeconfig to the repository root. |
| `ansible/install-dependencies.yml` | Installs Helm, MetalLB, NGINX Ingress, and an optional demo NGINX ingress app. |
| `ansible/install-argocd.yml` | Installs Argo CD and exposes `argocd-server` through a LoadBalancer service. |

Ansible inventory:

```text
ansible/inventory.ini
```

Shared variables:

```text
ansible/group_vars/all.yml
```

## Project Documentation

Full project documentation is available at:

```text
docs/project-documentation.md
```

Step-by-step installation tutorial:

```text
docs/tutorial-install-k3s-cluster.md
```

## Access The Cluster

After `install-k3s.yml` completes, use the generated kubeconfig:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get nodes -o wide
```

Expected node status:

```text
k3s-master     Ready   control-plane   192.168.56.33
k3s-worker-1   Ready   worker          192.168.56.34
k3s-worker-2   Ready   worker          192.168.56.35
```

## LoadBalancer IPs

MetalLB is configured by `ansible/install-dependencies.yml` with this address range:

```text
192.168.56.65-192.168.56.94
```

Example host entries:

```text
192.168.56.65 nginx.superapps.lab
192.168.56.66 argocd.superapps.lab
```

Add them to your host `/etc/hosts` only after confirming the actual LoadBalancer IPs:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get svc -A
```

## Useful Commands

Check nodes from the master:

```bash
vagrant ssh k3s-master -c "sudo k3s kubectl get nodes -o wide"
```

Check all pods:

```bash
vagrant ssh k3s-master -c "sudo k3s kubectl get pods -A -o wide"
```

Check K3s services:

```bash
vagrant ssh k3s-master -c "systemctl status k3s --no-pager"
vagrant ssh k3s-worker-1 -c "systemctl status k3s-agent --no-pager"
vagrant ssh k3s-worker-2 -c "systemctl status k3s-agent --no-pager"
```

## Troubleshooting

If `kubectl` from the host points to the wrong cluster, always pass the local kubeconfig explicitly:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get nodes
```

If nodes are `NotReady` after a VM IP change, verify the K3s node IPs and worker `K3S_URL` values in the Ansible variables:

```text
ansible/inventory.ini
ansible/group_vars/all.yml
```

If pods are stuck after stopping and starting VMs, restart the K3s services:

```bash
vagrant ssh k3s-master -c "sudo systemctl restart k3s"
vagrant ssh k3s-worker-1 -c "sudo systemctl restart k3s-agent"
vagrant ssh k3s-worker-2 -c "sudo systemctl restart k3s-agent"
```
