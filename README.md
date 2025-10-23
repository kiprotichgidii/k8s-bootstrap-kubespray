# Set Up HA Kubernetes Cluster on Ubuntu 24.04 with Kubespray

Kubespray is an open source, production-grade automation tooling for the deployment and management of highly available Kubernetes clusters on Linux operating systems. It is based on Ansible, and it avails to you full lifecycle management of Kubernetes clusters running in Cloud, on-premise, or hybrid environments. In this guide, we'll cover all the necessary steps when deploying a Highly Available (HA) Kubernetes cluster on Ubuntu 24.04 using Kubespray.

## Why Kubespray for Kubernetes Deployment?

Kubespray delivers the following capabilities:

- Support for multiple clouds

- A highly customizable Kubernetes deployment options

- Native support for HA Kubernetes control plane

- Enjoys an active open-source community

## Setup Pre-requisites

For this installation you will need:

1. Ubuntu 24.04 Machines - VMs or Baremetal
   - At least 3 master nodes and optionally worker nodes

   - SSH access to all nodes

   - All nodes must have unique hostnames and static IPs

   - Internet access on all machines

2. Workstation (Ansible Control Node): You also need a separate machine, this can be your local machine with:

   - Git

   - Python 3.8+

   - Ansible 2.14+

   - Access to the Kubernetes nodes via SSH

   - Internet Access

### Step 1: Prepare Kubernetes Nodes

In this article, we are going to use the following machine configurations - The control plane nodes will be used as worker nodes.

| Server IP     | Node Role                       | OS Version   |
| ------------- | ------------------------------- | ------------ |
| 192.168.20.18 | Control plane (and Worker Node) | Ubuntu 24.04 |
| 192.168.20.19 | Control plane (and Worker Node) | Ubuntu 24.04 |
| 192.168.20.20 | Control plane (and Worker Node) | Ubuntu 24.04 |


However, it's also possible to include separate, dedicated worker nodes. For example:

| Server IP     | Node Role   | OS Version   |
| ------------- | ----------- | ------------ |
| 192.168.20.21 | Worker Node | Ubuntu 24.04 |
| 192.168.20.22 | Worker Node | Ubuntu 24.04 |
| 192.168.20.23 | Worker Node | Ubuntu 24.04 |
| 192.168.20.24 | Worker Node | Ubuntu 24.04 |


Ensure each node has:

- Static IP addresses configured

- SSH access enabled

- Static hostname (if not managing using Kubespray), set using hostnamectl command.

Also disable swap on all nodes:
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Finally update packages on all nodes:
```bash
sudo apt update && sudo apt upgrade -y
[ -e /var/run/reboot-required ] && sudo reboot
```
### Step 2: Configure Ansible Control Node

Login to your Ansible control node, and install the following Python packages.This can be any OS:

#### Debian / Ubuntu:
```bash
sudo apt update
sudo apt install git python3-venv python3-full
sudo apt remove ansible ansible-core 2>/dev/bull
```
#### Rocky / AlmaLinux 9:
```bash
sudo dnf -y install git python3.11
```
Then create a Virtual Environment for Kubespray:

#### Debian / Ubuntu:
```bash
VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
python3 -m venv $VENVDIR
```
#### Rocky / AlmaLinux 9::
```bash
python3.11 -m venv $VENVDIR
```
Then clone the Kubespray Repository
```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
```
Activate Virtual Environment:
```bash
source $VENVDIR/bin/activate
```
Switch into kubespray directory:
```bash
cd kubespray
```
Check out to the relevant tag - See Kubespray releases page on Github:
```bash
git checkout <TAG> # Example  git checkout v2.29.0
```
Install required dependencies defined in the requirements.txt file:
```bash
pip install --upgrade pip
pip install -U -r requirements.txt
```
If the installation fails, try adding the --break-system-packages flag.

Verify version of Ansible installed:
```bash
$ ansible --version
ansible [core 2.16.14]
  config file = /root/kubespray/ansible.cfg
  configured module search path = ['/root/kubespray/library']
  ansible python module location = /usr/local/lib/python3.12/dist-packages/ansible
  ansible collection location = /root/.ansible/collections:/usr/share/ansible/collections
  executable location = /usr/local/bin/ansible
  python version = 3.12.3 (main, Feb  4 2025, 14:48:35) [GCC 13.3.0] (/usr/bin/python3)
  jinja version = 3.1.4
  libyaml = True
```
Copy the sample inventory:
```bash
cp -rfp inventory/sample inventory/mycluster
```
Create inventory file to reflect your topology. If you want to separate control plane nodes from worker nodes, make the necessary adjustments; otherwise, if they are shared, you can leave the configuration as is.
```bash
vim inventory/mycluster/hosts.yaml
```
#### Example configuration for shared nodes:
```yaml
all:
  hosts:
    k8smas01:
      ansible_host: 192.168.20.18
      ip: 192.168.20.18
      access_ip: 192.168.20.18
    k8smas02:
      ansible_host: 192.168.20.19
      ip: 192.168.20.19
      access_ip: 192.168.20.19
    k8smas03:
      ansible_host: 192.168.20.20
      ip: 192.168.20.20
      access_ip: 192.168.20.20
  children:
    kube_control_plane:
      hosts:
        k8smas01:
        k8smas02:
        k8smas03:
    kube_node:
      hosts:
        k8smas01:
        k8smas02:
        k8smas03:
    etcd:
      hosts:
        k8smas01:
        k8smas02:
        k8smas03:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```
If you are using the Jump server/Bastion host to access Kubernetes nodes, change IPs accordingly.

#### Example Configuration for separate master nodes and worker nodes:

In most cases, you’ll be updating the kube_node group to include additional worker nodes.
```yaml
all:
  hosts:
    k8smas01:
      ansible_host: 192.168.20.18
      ip: 192.168.20.18
      access_ip: 192.168.20.18
    k8smas02:
      ansible_host: 192.168.20.19
      ip: 192.168.20.19
      access_ip: 192.168.20.19
    k8smas03:
      ansible_host: 192.168.20.20
      ip: 192.168.20.20
      access_ip: 192.168.20.20
    
    k8snode01:
      ansible_host: 192.168.20.21
      ip: 192.168.20.21
      access_ip: 192.168.20.21
    k8snode02:
      ansible_host: 192.168.20.22
      ip: 192.168.20.22
      access_ip: 192.168.20.22
  children:
    kube_control_plane:
      hosts:
        k8smas01:
        k8smas02:
        k8smas03:
    kube_node:
      hosts:
        k8snode01:
        k8snode02:
    etcd:
      hosts:
        k8smas01:
        k8smas02:
        k8smas03:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```
### Step 3: Customizing Kubernetes cluster settings (Optional)

Edit k8s-cluster.yml file to customize cluster settings before deploy:
```bash
vim inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
```
Enable strict ARP:
```yaml
kube_proxy_strict_arp: true
```
Enable nodelocal dns cache
```yaml
enable_nodelocaldns: true
```
Setting network plugin:
```yaml
kube_network_plugin: cilium
```
Kubernetes internal network for services:
```yaml
kube_service_addresses: 10.233.0.0/18
```
Internal Pods network:
```yaml
kube_pods_subnet: 10.233.64.0/18
```
Container runtime:
```yaml
container_manager: containerd
```
Automatically renew K8S control plane certificates (monthly):
```yaml
auto_renew_certificates: false
```
#### Additional Addons

Edit the addons.yml file to adjust customizations for additional services
```bash
vim inventory/cluster1/group_vars/k8s_cluster/addons.yml
```
For example:

#### Kube VIP ( Configure this for HA Control Plane)
```yaml
# Kube VIP
kube_vip_enabled: true
kube_vip_arp_enabled: true
kube_vip_controlplane_enabled: true
kube_vip_address: 192.168.20.10 # Set correct virtual IP to be used
loadbalancer_apiserver:
  address: "{{ kube_vip_address }}"
  port: 6443
```
Optionally, you can configure an A record in your DNS server to map the kube_vip_address IP to your Kubernetes API endpoint DNS name.
For example:
```bash
192.168.20.10 k8sapi.cloudspinx.com
```
Enable Kubernetes Dashboard:
```yaml
dashboard_enabled: false
```
Metrics Server deployment
```yaml
metrics_server_enabled: true
```
Cert manager deployment
```yaml
cert_manager_enabled: true
```
If you want to disable Kubespray setting the hostname to inventory_hostname, edit:
```bash
$ vim roles/bootstrap-os/defaults/main.yml
override_system_hostname: true # Set false to use manually set hostnames
```
### Step 4: Deploy Kubernetes cluster

If using LB, configure it in inventory/cluster1/group_vars/all/all.yml before you start cluster installation.
```yaml
## External LB example config
apiserver_loadbalancer_domain_name: "elb.some.domain"
  loadbalancer_apiserver:
    address: 1.2.3.4
    port: 1234
```
Below is a list of commonly used Kubespray options:

| Option            | Description                                                                             |
| ----------------- | --------------------------------------------------------------------------------------- |
| `-i` (Inventory)  | Specifies the inventory file for hosts.                                                 |
| `-b`, `--become`  | Runs the playbook with **sudo (privileged)** access on the target nodes.                |
| `-v`, `--verbose` | Increases verbosity of the output. Use multiple `v` for more verbosity (`-vv`, `-vvv`). |
| `--private-key`   | Specifies the **private SSH key** for authentication.                                   |
| `--limit`         | Limits the playbook run to **specific hosts or groups**.                                |
| `--extra-vars`    | Passes **extra variables** that override default configurations.                        |
| `--check`         | Runs the playbook in **“dry-run” mode** without applying changes.                       |


Initiate Kubernetes cluster deployment on Ubuntu 24.04 using Kubespray:
```bash
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml
```
The process may take more than 10 minutes depending on your cluster size.

### Step 5: Access Kubernetes Cluster

After successful execution of the playbook, copy kubeconfig from one of the control-plane nodes:
```bash
cd ../
NODE1=192.168.20.18
scp root@$NODE1:/etc/kubernetes/admin.conf ./k8s-kubeconfig
```
If you have Kube VIP enabled, you will see https://lb-apiserver.kubernetes.local:6443
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJUkRIT1QxVnRxVzR3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TlRBME1UWXhNRE16TkRSYUZ3MHpOVEEwTVRReE1ETTRORFJhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUUNtZHFDbXJhN29KMGoxSE1MaFRlS3hxUlI1M01XWDNnQ2hMa0dtVlpMd3BUWjBZbXVsMXEwU2dMdm4KdCt0dXJLV3V5VjBLY2lwbVpYZXJQT2tNTElQK0lBVWpXTmUwWEhEbVIyQjlLZnpWWVJFVE9BUTdaSC9XNzhlMAp4Y2kwcjVZU3Y0V0V5QjUyUGRkbXdRcXMzeE52RXJveDlPbzBwa1krekVlNFNtWmI2NXRZNmpTZkRmZDZUcVRlCmhVUjI2eUlQTHJYVWVTNk1yV3FZWmJyOGZIZ3ZkaGtXcURLUkFlU0VCOHBFb2lqcnlpZzBVanc1QURheXFoL2EKNmowODNIWVZ2d3VsR0ZjT3hsNko4QXcvWTV4bmRidDNnQ2t6d3hjSU1ybkdHWitua1VxS1dNQkoxWEpRNWR5ZQorOHorOFJIVWJPZ1dJYzNncU5mdFVwQzZHV24zQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJTNTVSS0VNa1VOb1p4a0lidEpNQ2NNRFRiTWZ6QVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQkIvYXpPNmRjKwpRK3pzK3pYbjd1dUVBbVBXVnl1c1BVZGw0c2dxSVNjMjh2OENndGhsekNNSndqcjdTNm8xRGlRNGtCOWd0clViCnkvUVJZYU1tb0tCR1VVWGpod1FtV1YyN0Juc1kxOHd4RG1wd3EySHVYQktjUXUzUTRKQ1BaWVFicDV1Yyt5L2EKT2llanhiaEdRaUxnQi9tMEdXN3hXRGdiZWx0UXhWbm4rYXo2UVZ2M2J1VkIxWGZwQllPRlZEdU1XZFAzcDQrdQpGNGxITDhxaXdNbm1MeG5hYTJMMGpxcFFreXYvRGF5dkdzQTZxTjdrVjZQT3VRU0ROYXF3VmpOV2h3RTVnRmtsCjZQcnN4NkR0azhIU1ZpQzliaFdUVStxb0xJUjArLy9HUHFpMVlLb1crSytwMDJrWXhJeEZnd3JUdWZMZVBJM0YKVHR3Wm1FaDhuY0UvCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://lb-apiserver.kubernetes.local:6443
  name: cluster.local
contexts:
- context:
    cluster: cluster.local
    user: kubernetes-admin
  name: kubernetes-admin@cluster.local
current-context: kubernetes-admin@cluster.local
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURLVENDQWhHZ0F3SUJBZ0lJTTI0M2tHOC9HYUl3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TlRBME1UWXhNRE16TkRSYUZ3MHlOakEwTVRZeE1ETTRORFZhTUR3eApIekFkQmdOVkJBb1RGbXQxWW1WaFpHMDZZMngxYzNSbGNpMWhaRzFwYm5NeEdUQVhCZ05WQkFNVEVHdDFZbVZ5CmJtVjBaWE10WVdSdGFXNHdnZ0VpTUEwR0NTcUdTSWIzRFFFQkFRVUFBNElCRHdBd2dnRUtBb0lCQVFEQ1BORFoKNlNSYkV0T0FBM3BDY0xndWNVVXhWMmFYUHQrM3JpRmpsRVBmQjNGeVROWGhtZGpWRWpaRjIvVTRSS01LK1FPZwoyc0hXTGxzTjBjb0dYWnJRUTZGWFh4bUdNdDRCdUFNZWFQQ1pCRWZtcnlJeWlwQVNrSzVoeklPMmkrUTRGZlRSCk9IK2RIUGxGV3BWWThSQndXZ2t1R25nVzhLUHd4MjdoczlPOGNkWkFxZEFLZUMrT0UxaFdnalY4UzAyb2Y4TzMKZFRMRUNOYW9abHJ3T2x1Z1FxejFKaEZsbEZTR0hCbkFsT1J1ZlY0MWdqMWFtU0NyU1RRWGFnWU9VbE8yU0tZSQpkdXV0bThWS2xUQU8vRldSRyt6RWFMUU1pTkNpbUpGa3cvTC83NEJURFhySUR4V0Njd0MrYU5aNEhEQjFLc01LCm1rckhqK3B2QmkwZURuTG5BZ01CQUFHalZqQlVNQTRHQTFVZER3RUIvd1FFQXdJRm9EQVRCZ05WSFNVRUREQUsKQmdnckJnRUZCUWNEQWpBTUJnTlZIUk1CQWY4RUFqQUFNQjhHQTFVZEl3UVlNQmFBRkxubEVvUXlSUTJobkdRaAp1MGt3Snd3Tk5zeC9NQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUJBUUNGMFcxa3ZqNmpKVStiK2hranlWSkZkTXNLCm51bVJuVGt0eTROVjg3aDR2cFpJWThnOHFBLzNrcVJGUkNwN2hydlgyd04rbU1sSWhMVDFmYXJSZTl5SFVJaEUKRk1HeGFvdURvSER1SkZ6L05KTnZyQk9KMTkrQkwwRlJSQVplQkpFOUdsVGYzVGxPZTRYaG5NTjFvOWVqTXVPSwpUdWh1VEdXSHVFaVlCMmhOZm9ON1FzbWhiMlVxd0xWYkdEMTZzbDZIejdRaFhSNlNVaDNiNExHWkhqUmpCUStaCmxzUDNseHVQNk1SRWFONCsrVWtmQ2hOUVpySlNqSXovVG11MWtXOTVwM3hjVHM4WWR5b0hqY294eCtLY1dSdkoKMTFZS0JMem9oRzQzN2M4UHZqckRjaGUrV2RYb0NyaFZnKzkzbUF5WEUvWktxT3ZtRDNsVUo5R1RKdGpiCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBd2p6UTJla2tXeExUZ0FONlFuQzRMbkZGTVZkbWx6N2Z0NjRoWTVSRDN3ZHhja3pWCjRablkxUkkyUmR2MU9FU2pDdmtEb05yQjFpNWJEZEhLQmwyYTBFT2hWMThaaGpMZUFiZ0RIbWp3bVFSSDVxOGkKTW9xUUVwQ3VZY3lEdG92a09CWDAwVGgvblJ6NVJWcVZXUEVRY0ZvSkxocDRGdkNqOE1kdTRiUFR2SEhXUUtuUQpDbmd2amhOWVZvSTFmRXROcUgvRHQzVXl4QWpXcUdaYThEcGJvRUtzOVNZUlpaUlVoaHdad0pUa2JuMWVOWUk5Cldwa2dxMGswRjJvR0RsSlR0a2ltQ0hicnJadkZTcFV3RHZ4VmtSdnN4R2kwRElqUW9waVJaTVB5LysrQVV3MTYKeUE4VmduTUF2bWpXZUJ3d2RTckRDcHBLeDQvcWJ3WXRIZzV5NXdJREFRQUJBb0lCQUhHQkNtYWNoK00wZ0NWcApZdE5hZlRhZWVGbWFBbGhWcEhQNHJJZzlSdUFZd0dHVHB0UjdpNnNQUm1uU1hGenlOdmlkaFZKRkkwcGVzbFRFCkNETnFGYUtvTXFzVTVweDJNeWQ3K1U2Vzhpbm94MzkxVGgyTXZSNHNMOHIwc085R2xpbDBJeWp6eEJieXJIT3IKdUdST0VsWWxOd0lhODV3c0tSRDE2Y1M0eWYxdTNXU3RTaWh1ZSt4ZWpDOXpQY0RrMkJMZVdwUDBRUGlhemNEKwpUeTkwKzFEUGV1UDFtV1pOMENKUHNmMHNyeGN1WlBtcXc2b1hDY1lOUkNncE0yQlg1YjUxSHBpdkV3SVBIRjRJCk9XUDdQaERLSTM0djhzM3dwWUFGQlI4NWs3ZDhxcy9aa1pKOE40a1Nmc1ZqbTNrUjZpSUhkZVhlcmZHeUhwMmkKWkNkdmxGRUNnWUVBdzB4NGxxNlE1QXNHWmw4Wk9ldEhWMnlCclRUZnBaMTM1K0VCQm54UmRrTFY2VEkrczJBKwoyZmc2a3JMN25qSWg1dFR6clhrUnRNeHZBSGJKM1NNNVVndTJManJET0s1OCt1czBmYzNuWDF4MjNzWGplYWFWCk5zSml2OVBNMXFXMFNad0hQa3o1d1ZvSjNMVTMvZWNEVW1Xd01CLzVTRFF3eHJxOWZqRUxpSDhDZ1lFQS9wdnAKTXRtZHpWa0ttaFBQWWZYUXNQTk8zSktZVEpycHZld3EyRnlLRG5GMENoN1hKNXZlbmZMRjBXRkpEck4zaFVFRgpURjRWeHdjYlVsVm9mS2NVZ0NFN09HTllkTjU0enlOQ1dpTHhpSGJqUW5kVUJDUGZEbTVCSW50UEoycEFjR2YvCkc0OE5OcWdqeUUzUDlkajBOdVIzVlJUd1J2b0ZGdFRURWJtdW9aa0NnWUVBcDV2WW5sRkJEa1diLzMyOFU2WGwKdTFUblVmUmZ3RzROZXhieTMxTVFRck9IakRSUDlYZ3pXTFFkNk1ydEFVNjdJN1U5VUhMb1RFZHJPSFc2Tnl4RQp4SEpDcnhoRmRUN2pDaUdVRWlnRld5VXE2M1BnRHdaMVp1S2JCMURKcXFuWnVaYkw3Sjc1ZGdSRkZJTCtnOHlnCllEWGZhTjMzL2d5MGs4bXVXVC9VU3hjQ2dZQlhPNmZjYWo3c3VsTXRreGY4b2pJTVRuQjRsaWxrSmJkc0FOeDEKSU0rVVB6N1lzTlJhbDhiZ0t1dW4zME1lckZLSTcwd1hiQ3pkOGd0a1hDcmVlb2hGbGgwcUpxK0o2eWROSVBGOAozSGdRbjFzaHpLeVdkb3ZYNytLVkk5WnMxTFNiVHFaVEZPSWNGZU9jbnp4ZktTUVRJcGZZS01KaUx3dExWVU96CjBRQ0tFUUtCZ0NtQ1B3VkxlQ3p5STVpdHZaK0RGaHB4K0VTMHBYZ2VsZVdad1NNS0JKa1VOWjFCdElZRFludFQKNDBreFJGT2RuNUk1bUhNUWRyM3FrWi8zUmorM0tVaU0xYkhOaUJETy9BOWFobndTZEt0b2gwNlprbE0zZzhaUgptQzFGbUtGOWlDditPTE5Dd1hEcGc4NmtMcU91Q1BzRStVRUZTY3pTelNLb0EyTXN6TkszCi0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```
Add mapping for VIP to hostname in /etc/hosts file.
```bash
$ sudo vim /etc/hosts
192.168.20.10 lb-apiserver.kubernetes.local
```
Set copied kubeconfig in KUBECONFIG env variable:
```bash
export KUBECONFIG=./k8s-kubeconfig
```
Test access to your newly created kubernetes cluster using kubectl:
```bash
$ kubectl get nodes
NAME    STATUS   ROLES           AGE   VERSION
node1   Ready    control-plane   18m   v1.31.4
node2   Ready    control-plane   18m   v1.31.4
node3   Ready    control-plane   17m   v1.31.4
```
### Uninstalling Kubernetes using Kubespray

Kubespray offers the reset.yml playbook which is used to uninstall Kubernetes from the running nodes.
```bash
ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root reset.yml
```
Confirm with 'yes' when prompted to proceed with cluster deletion

#### Remove leftovers

Then run the following command to clean up any leftovers (if any):
```bash
ansible all -i inventory/cluster1/hosts.yaml -m shell -a \
'rm -rf /etc/kubernetes /etc/cni /opt/cni /var/lib/etcd /var/lib/cni /var/lib/kubelet /var/run/kubernetes /var/lib/containerd /etc/systemd/system/kubelet.service.d /etc/containerd'
```
#### References:

- [Kubespray github](https://github.com/kubernetes-sigs/kubespray/tree/master)

- [Kubespray HA documentation](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/ha-mode.md)

- [Adding / Removing nodes](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/operations/nodes.md)