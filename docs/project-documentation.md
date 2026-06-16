# K3s Cluster Project Documentation

## Overview

This project builds a local Kubernetes lab using Vagrant, Ubuntu 24.04, K3s, MetalLB, NGINX Ingress, and Argo CD.

The lab is intended for learning and practicing:

- Kubernetes cluster installation
- multi-node K3s operations
- local LoadBalancer networking with MetalLB
- ingress routing with NGINX Ingress Controller
- GitOps workflows with Argo CD
- repeatable infrastructure automation with Ansible

The `Vagrantfile` creates the virtual machines. The Ansible playbooks install and configure the Kubernetes stack.

## Architecture

```text
Host machine
  |
  | private network 192.168.56.0/24
  |
  +-- k3s-master    192.168.56.33   K3s server / control plane
  +-- k3s-worker-1  192.168.56.34   K3s agent / worker
  +-- k3s-worker-2  192.168.56.35   K3s agent / worker

Kubernetes services
  |
  +-- MetalLB LoadBalancer range: 192.168.56.65-192.168.56.94
  +-- NGINX Ingress Controller
  +-- Argo CD
```

## Repository Layout

```text
.
├── Vagrantfile
├── README.md
├── kubeconfig-k3s-cluster.yaml
├── ansible/
│   ├── ansible.cfg
│   ├── inventory.ini
│   ├── group_vars/
│   │   └── all.yml
│   ├── install-k3s.yml
│   ├── install-dependencies.yml
│   ├── install-argocd.yml
│   └── README.md
└── docs/
    ├── project-documentation.md
    └── tutorial-install-k3s-cluster.md
```

## Node Inventory

| Node | Role | IP | Service |
| --- | --- | --- | --- |
| `k3s-master` | Control plane | `192.168.56.33` | `k3s` |
| `k3s-worker-1` | Worker | `192.168.56.34` | `k3s-agent` |
| `k3s-worker-2` | Worker | `192.168.56.35` | `k3s-agent` |

Default VM credentials:

```text
username: vagrant
password: vagrant
```

## Prerequisites

Install these tools on the host machine:

- Vagrant
- VMware Desktop or VirtualBox provider
- Ansible
- `sshpass` if using password-based SSH
- `kubectl` for host-side cluster access

Example macOS setup:

```bash
brew install ansible sshpass kubectl
```

Optional:

```bash
vagrant plugin install vagrant-disksize
```

## Provisioning Workflow

### 1. Start VMs

```bash
vagrant up
```

Check VM status:

```bash
vagrant status
```

Expected state:

```text
k3s-master     running
k3s-worker-1   running
k3s-worker-2   running
```

### 2. Install K3s

```bash
cd ansible
ansible-playbook install-k3s.yml
```

The playbook:

- installs the K3s server on `k3s-master`
- installs K3s agents on both worker nodes
- joins workers to the master
- labels workers
- creates the `dev`, `qa`, and `production` namespaces
- writes the host kubeconfig to `kubeconfig-k3s-cluster.yaml`

For a clean reinstall on existing VMs:

```bash
ansible-playbook install-k3s.yml -e k3s_reset_cluster=true
```

### 3. Install Cluster Dependencies

```bash
ansible-playbook install-dependencies.yml
```

The playbook installs:

- Helm
- MetalLB
- NGINX Ingress Controller
- optional demo NGINX app and ingress

Default MetalLB range:

```text
192.168.56.65-192.168.56.94
```

To skip the demo app:

```bash
ansible-playbook install-dependencies.yml -e dependency_demo_enabled=false
```

### 4. Install Argo CD

```bash
ansible-playbook install-argocd.yml
```

The playbook:

- creates the `argocd` namespace
- installs the upstream stable Argo CD manifest
- waits for Argo CD components
- exposes `argocd-server` as a LoadBalancer
- prints the initial admin password

Optional Argo CD CLI installation on the master:

```bash
ansible-playbook install-argocd.yml -e argocd_install_cli=true
```

## Host Access

Use the generated kubeconfig from the repository root:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get nodes -o wide
```

Expected output:

```text
NAME           STATUS   ROLES           INTERNAL-IP
k3s-master     Ready    control-plane   192.168.56.33
k3s-worker-1   Ready    worker          192.168.56.34
k3s-worker-2   Ready    worker          192.168.56.35
```

## Networking

The cluster uses the Vagrant private network:

```text
192.168.56.0/24
```

Node IPs are reserved for the VMs:

```text
192.168.56.33  k3s-master
192.168.56.34  k3s-worker-1
192.168.56.35  k3s-worker-2
```

LoadBalancer IPs are reserved for MetalLB:

```text
192.168.56.65-192.168.56.94
```

Default service allocation:

| Service | Hostname | LoadBalancer IP |
| --- | --- | --- |
| NGINX Ingress Controller | `nginx.superapps.lab` | `192.168.56.65` |
| Argo CD Server | `argocd.superapps.lab` | `192.168.56.66` |

Example local DNS entries:

```text
192.168.56.65 nginx.superapps.lab
192.168.56.66 argocd.superapps.lab
```

Confirm the actual assigned service IPs before editing `/etc/hosts`:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get svc -A
```

## Installed Components

### K3s

K3s is installed without the default Traefik and ServiceLB components:

```text
--disable traefik
--disable servicelb
```

This allows the project to use:

- MetalLB for LoadBalancer services
- NGINX Ingress Controller for ingress traffic

### MetalLB

MetalLB provides LoadBalancer IPs on the local Vagrant network.

Default resources:

- namespace: `metallb-system`
- IPAddressPool: `homelab-pool`
- L2Advertisement: `homelab-l2`

Check status:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get pods -n metallb-system
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get ipaddresspools -n metallb-system
```

### NGINX Ingress Controller

NGINX Ingress Controller is installed through Helm.

Check status:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get pods -n ingress-nginx
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get svc -n ingress-nginx
```

### Demo Application

The dependency playbook can deploy a demo NGINX app:

- namespace: `apps`
- deployment: `nginx-demo`
- service: `nginx-demo`
- ingress: `nginx-demo-ingress`
- host: `nginx.superapps.lab`

Test:

```bash
curl -H "Host: nginx.superapps.lab" http://192.168.56.65
```

Use the actual NGINX Ingress external IP if it differs from `192.168.56.65`.

### Argo CD

Argo CD is installed in the `argocd` namespace.

Check status:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get pods -n argocd
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get svc -n argocd
```

Get the initial admin password manually:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

Login:

```text
username: admin
password: <initial-admin-password>
```

## Daily Operations

Start the lab:

```bash
vagrant up
```

Stop the lab:

```bash
vagrant halt
```

Restart the lab:

```bash
vagrant reload
```

Check cluster health:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get nodes -o wide
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get pods -A -o wide
```

Check services directly from master:

```bash
vagrant ssh k3s-master -c "sudo k3s kubectl get nodes -o wide"
vagrant ssh k3s-master -c "sudo k3s kubectl get pods -A -o wide"
```

Restart K3s services:

```bash
vagrant ssh k3s-master -c "sudo systemctl restart k3s"
vagrant ssh k3s-worker-1 -c "sudo systemctl restart k3s-agent"
vagrant ssh k3s-worker-2 -c "sudo systemctl restart k3s-agent"
```

## Rebuild Procedure

Use this flow when you want a fully clean lab:

```bash
vagrant destroy -f
vagrant up

cd ansible
ansible-playbook install-k3s.yml
ansible-playbook install-dependencies.yml
ansible-playbook install-argocd.yml
```

Use this flow when the VMs should stay but K3s must be reset:

```bash
cd ansible
ansible-playbook install-k3s.yml -e k3s_reset_cluster=true
ansible-playbook install-dependencies.yml
ansible-playbook install-argocd.yml
```

## Troubleshooting

### Host kubectl points to the wrong cluster

Always pass the local kubeconfig explicitly:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get nodes
```

### Node is NotReady

Check the service:

```bash
vagrant ssh k3s-master -c "systemctl status k3s --no-pager"
vagrant ssh k3s-worker-1 -c "systemctl status k3s-agent --no-pager"
vagrant ssh k3s-worker-2 -c "systemctl status k3s-agent --no-pager"
```

Check node IPs:

```bash
vagrant ssh k3s-master -c "hostname -I"
vagrant ssh k3s-worker-1 -c "hostname -I"
vagrant ssh k3s-worker-2 -c "hostname -I"
```

The private IPs must match `ansible/inventory.ini`.

### Pods are stuck after VM restart

Restart K3s services:

```bash
vagrant ssh k3s-master -c "sudo systemctl restart k3s"
vagrant ssh k3s-worker-1 -c "sudo systemctl restart k3s-agent"
vagrant ssh k3s-worker-2 -c "sudo systemctl restart k3s-agent"
```

Then check:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get pods -A -o wide
```

### LoadBalancer external IP is pending

Check MetalLB:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get pods -n metallb-system
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get ipaddresspools -n metallb-system
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get l2advertisements -n metallb-system
```

Confirm that the requested LoadBalancer IP is inside:

```text
192.168.56.65-192.168.56.94
```

## Configuration Reference

Main variables are stored in:

```text
ansible/group_vars/all.yml
```

Important variables:

| Variable | Description |
| --- | --- |
| `k3s_master_ip` | Kubernetes API endpoint IP. |
| `k3s_disable_components` | K3s bundled components disabled during install. |
| `k3s_reset_cluster` | Whether to uninstall existing K3s before reinstalling. |
| `metallb_address_range` | MetalLB LoadBalancer IP range. |
| `metallb_address_ranges` | List of MetalLB LoadBalancer ranges applied to the pool. |
| `ingress_nginx_loadbalancer_ip` | Fixed LoadBalancer IP for the NGINX Ingress Controller. |
| `dependency_demo_enabled` | Whether to deploy the demo NGINX app. |
| `argocd_service_type` | Service type for `argocd-server`. |
| `argocd_loadbalancer_ip` | Fixed LoadBalancer IP for `argocd-server`. |
| `argocd_url_scheme` | Public URL scheme for Argo CD, currently `http`. |
| `argocd_install_cli` | Whether to install the Argo CD CLI on the master. |
