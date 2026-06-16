# Tutorial: Install a Local K3s Cluster with Vagrant and Ansible

This tutorial walks through installing a local three-node K3s cluster using this repository.

The final cluster will contain:

- one K3s control plane node
- two K3s worker nodes
- host-side kubeconfig access
- optional cluster add-ons such as MetalLB, NGINX Ingress, and Argo CD

## 1. Cluster Topology

The lab uses three Ubuntu 24.04 virtual machines created by Vagrant.

| Node | Role | IP |
| --- | --- | --- |
| `k3s-master` | Control plane | `192.168.56.33` |
| `k3s-worker-1` | Worker | `192.168.56.34` |
| `k3s-worker-2` | Worker | `192.168.56.35` |

The nodes use the Vagrant private network:

```text
192.168.56.0/24
```

## 2. Prerequisites

Install the required tools on your host machine:

- Vagrant
- VMware Desktop or VirtualBox provider
- Ansible
- `sshpass`
- `kubectl`

On macOS, you can install the CLI tools with Homebrew:

```bash
brew install ansible sshpass kubectl
```

Optional plugin for VM disk sizing:

```bash
vagrant plugin install vagrant-disksize
```

## 3. Clone or Open the Project

Open the project directory:

```bash
cd /Users/budiart/Documents/vagrant/k3s-cluster
```

The important files are:

```text
Vagrantfile
ansible/install-k3s.yml
ansible/inventory.ini
ansible/group_vars/all.yml
```

## 4. Start the Vagrant VMs

Start all nodes:

```bash
vagrant up
```

Check the VM status:

```bash
vagrant status
```

Expected result:

```text
k3s-master     running
k3s-worker-1   running
k3s-worker-2   running
```

You can also SSH into any node:

```bash
vagrant ssh k3s-master
vagrant ssh k3s-worker-1
vagrant ssh k3s-worker-2
```

## 5. Review the Ansible Inventory

The inventory is located at:

```text
ansible/inventory.ini
```

It defines the three Vagrant nodes:

```ini
[k3s_master]
k3s-master ansible_host=192.168.56.33 k3s_node_ip=192.168.56.33

[k3s_workers]
k3s-worker-1 ansible_host=192.168.56.34 k3s_node_ip=192.168.56.34
k3s-worker-2 ansible_host=192.168.56.35 k3s_node_ip=192.168.56.35
```

The default SSH credentials are:

```text
username: vagrant
password: vagrant
```

## 6. Install K3s with Ansible

Move into the Ansible directory:

```bash
cd ansible
```

Run the K3s installer playbook:

```bash
ansible-playbook install-k3s.yml
```

The playbook will:

- install the K3s server on `k3s-master`
- disable bundled Traefik and ServiceLB
- read the node token from the master
- install K3s agents on both workers
- join the workers to the cluster
- label worker nodes
- create the `dev`, `qa`, and `production` namespaces
- write kubeconfig to `../kubeconfig-k3s-cluster.yaml`

## 7. Verify the Cluster

Return to the project root:

```bash
cd ..
```

Use the generated kubeconfig:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get nodes -o wide
```

Expected result:

```text
NAME           STATUS   ROLES           INTERNAL-IP
k3s-master     Ready    control-plane   192.168.56.33
k3s-worker-1   Ready    worker          192.168.56.34
k3s-worker-2   Ready    worker          192.168.56.35
```

Check all namespaces:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get namespaces
```

You should see:

```text
dev
qa
production
```

## 8. Install Cluster Dependencies

After K3s is ready, install Helm, MetalLB, and NGINX Ingress:

```bash
cd ansible
ansible-playbook install-dependencies.yml
```

This playbook installs:

- Helm
- MetalLB `v0.15.2`
- NGINX Ingress Controller
- a demo NGINX app and ingress

MetalLB uses this LoadBalancer IP range:

```text
192.168.56.65-192.168.56.94
```

The default service IP assignment is:

```text
192.168.56.65 nginx.superapps.lab
192.168.56.66 argocd.superapps.lab
```

Check services:

```bash
cd ..
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get svc -A
```

If the demo app is enabled, add the NGINX Ingress IP to your host `/etc/hosts`:

```text
192.168.56.65 nginx.superapps.lab
```

Then test:

```bash
curl http://nginx.superapps.lab
```

To skip the demo app during dependency installation:

```bash
cd ansible
ansible-playbook install-dependencies.yml -e dependency_demo_enabled=false
```

## 9. Install Argo CD

Install Argo CD after the cluster dependencies are ready:

```bash
cd ansible
ansible-playbook install-argocd.yml
```

The playbook will:

- create the `argocd` namespace
- install the upstream stable Argo CD manifests
- expose `argocd-server` as a LoadBalancer
- print the initial `admin` password

Check the Argo CD service:

```bash
cd ..
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get svc -n argocd
```

If Argo CD receives `192.168.56.66`, add this entry to your host `/etc/hosts`:

```text
192.168.56.66 argocd.superapps.lab
```

Then open:

```text
http://argocd.superapps.lab
```

Login with:

```text
username: admin
password: <initial-password-from-playbook-output>
```

If needed, get the password manually:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

## 10. Daily Commands

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

Check node health:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get nodes -o wide
```

Check pod health:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get pods -A -o wide
```

Check from the master node:

```bash
vagrant ssh k3s-master -c "sudo k3s kubectl get nodes -o wide"
vagrant ssh k3s-master -c "sudo k3s kubectl get pods -A -o wide"
```

## 11. Clean Reinstall

If you want to reinstall K3s while keeping the VMs:

```bash
cd ansible
ansible-playbook install-k3s.yml -e k3s_reset_cluster=true
ansible-playbook install-dependencies.yml
ansible-playbook install-argocd.yml
```

If you want to rebuild everything from scratch:

```bash
vagrant destroy -f
vagrant up

cd ansible
ansible-playbook install-k3s.yml
ansible-playbook install-dependencies.yml
ansible-playbook install-argocd.yml
```

## 12. Troubleshooting

### Host kubectl points to another cluster

Always use the local kubeconfig:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get nodes
```

### A node is NotReady

Check the K3s service:

```bash
vagrant ssh k3s-master -c "systemctl status k3s --no-pager"
vagrant ssh k3s-worker-1 -c "systemctl status k3s-agent --no-pager"
vagrant ssh k3s-worker-2 -c "systemctl status k3s-agent --no-pager"
```

Restart the services:

```bash
vagrant ssh k3s-master -c "sudo systemctl restart k3s"
vagrant ssh k3s-worker-1 -c "sudo systemctl restart k3s-agent"
vagrant ssh k3s-worker-2 -c "sudo systemctl restart k3s-agent"
```

### LoadBalancer external IP is pending

Check MetalLB:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get pods -n metallb-system
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get ipaddresspools -n metallb-system
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get l2advertisements -n metallb-system
```

Confirm that the MetalLB range does not overlap with node IPs:

```text
MetalLB: 192.168.56.65-192.168.56.94
Nodes:   192.168.56.33-192.168.56.35
```

## Summary

You now have a repeatable local Kubernetes lab:

- Vagrant creates the VMs
- Ansible installs K3s
- MetalLB provides local LoadBalancer IPs
- NGINX Ingress routes HTTP traffic
- Argo CD enables GitOps deployment workflows

This setup is useful for learning Kubernetes operations, testing manifests, and practicing platform engineering workflows locally.
