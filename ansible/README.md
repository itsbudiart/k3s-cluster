# K3s Cluster Ansible Playbooks

These playbooks install and repair the Vagrant K3s cluster with the current node IPs:

- `k3s-master`: `192.168.56.33`
- `k3s-worker-1`: `192.168.56.34`
- `k3s-worker-2`: `192.168.56.35`

## Install A Fresh Cluster

What `install-k3s.yml` does:

- installs the K3s server on the master node
- disables `traefik` and `servicelb`
- reads the cluster token automatically from the master node
- installs the K3s agent on worker nodes
- joins worker nodes to the master node
- writes kubeconfig to `../kubeconfig-k3s-cluster.yaml`
- labels worker nodes
- creates the `dev`, `qa`, and `production` namespaces
- validates that all nodes are `Ready`

Run from the repository root:

```bash
vagrant up
cd ansible
ansible-playbook install-k3s.yml
```

If an old cluster already exists and you want a clean reinstall:

```bash
ansible-playbook install-k3s.yml -e k3s_reset_cluster=true
```

After installation, access the cluster from the host:

```bash
KUBECONFIG=./kubeconfig-k3s-cluster.yaml kubectl get nodes -o wide
```

## Install Cluster Dependencies

What `install-dependencies.yml` does:

- installs Helm on the master node
- installs MetalLB `v0.15.2`
- configures the MetalLB address pool `192.168.56.65-192.168.56.94`
- installs NGINX Ingress Controller with Helm
- waits for the NGINX Ingress external IP
- optionally creates a demo NGINX app and ingress for `nginx.superapps.lab`

Run after the K3s cluster is ready:

```bash
cd ansible
ansible-playbook install-dependencies.yml
```

To skip the demo app:

```bash
ansible-playbook install-dependencies.yml -e dependency_demo_enabled=false
```

If you keep the demo app, add the printed ingress external IP to your host `/etc/hosts`, for example:

```text
192.168.56.65 nginx.superapps.lab
```

## Install Argo CD

What `install-argocd.yml` does:

- creates the `argocd` namespace
- installs the upstream stable Argo CD manifest
- waits for Argo CD deployments and statefulsets
- exposes `argocd-server` as a `LoadBalancer`
- waits for the external IP from MetalLB
- prints the initial `admin` password

Run after the K3s cluster is ready:

```bash
cd ansible
ansible-playbook install-argocd.yml
```

If you also want the Argo CD CLI installed on the master node:

```bash
ansible-playbook install-argocd.yml -e argocd_install_cli=true
```

After installation, open:

```text
https://<argocd-loadbalancer-ip>
```

The hostname configured in variables is:

```text
argocd.superapps.lab
```

Add it to your host `/etc/hosts` manually if you want to use the hostname.

## Repair Cluster Existing

`repair-k3s.yml` is still available for existing clusters when node IPs change or the containerd/CNI runtime gets stuck.

What the repair playbook does:

- updates the `k3s` and `k3s-agent` systemd overrides
- updates worker `K3S_URL` values to the current master IP
- reads the cluster token automatically from the master node
- regenerates the master serving certificate cache
- clears stale runtime/CNI state with `k3s-killall.sh`
- re-registers worker nodes with the current IPs
- labels worker nodes
- recycles pods that are not ready
- validates node and pod status

Run:

```bash
ansible-playbook repair-k3s.yml
```

If `sshpass` is not installed and you use the `vagrant` password, install it on the host first:

```bash
brew install sshpass
```

Alternatively, update the inventory to use the Vagrant private key.

## Options

Edit `group_vars/all.yml` to change behavior:

- `k3s_reset_cluster: false`
- `k3s_cleanup_runtime: true`
- `k3s_reregister_workers: true`
- `k3s_recycle_non_ready_pods: true`

To repair only the configuration without runtime or pod cleanup:

```bash
ansible-playbook repair-k3s.yml \
  -e k3s_cleanup_runtime=false \
  -e k3s_reregister_workers=false \
  -e k3s_recycle_non_ready_pods=false
```
