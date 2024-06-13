# Kubernetes Deployment with Kubespray: README

## Prerequisites

Before starting the installation, ensure that your systems meet the following requirements:

1. **Operating System:** Linux distributions like Ubuntu, CentOS, Debian, etc.
2. **SSH Access:** SSH access to all nodes.
3. **Sudo Access:** A user with passwordless sudo access.
4. **Python Installation:** Python version 3.5 or higher installed on all nodes.
5. **Ansible Installed:** Ansible must be installed on the machine you plan to use to manage the cluster.

## Installation Steps

### Step 1: Clone the Kubespray Repository

First, clone the Kubespray repository from GitHub:

```bash
git clone https://github.com/kubernetes-sigs/kubespray.git
cd kubespray
```

### Step 2: Install Dependencies

Install the Ansible dependencies:

```bash
VENVDIR=kubespray-venv
KUBESPRAYDIR=kubespray
python3 -m venv $VENVDIR
source $VENVDIR/bin/activate
cd $KUBESPRAYDIR
pip install -U -r requirements.txt
```

### Step 3: Configure the Nodes

Create a sample inventory file. This file will list all the nodes in your cluster:

```bash
cp -rfp inventory/sample inventory/mycluster
declare -a IPS=(192.168.61.10 192.168.61.11 192.168.61.12 192.168.61.13 192.168.61.14 192.168.61.15)
CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```

Then open the `inventory/mycluster/inventory.ini` file and add your node details. The structure of the file should be as follows:

```ini
all:
  hosts:
    node1:
      ansible_host: 192.168.61.10
      ip: 192.168.61.10
      access_ip: 192.168.61.10
    node2:
      ansible_host: 192.168.61.11
      ip: 192.168.61.11
      access_ip: 192.168.61.11
    node3:
      ansible_host: 192.168.61.12
      ip: 192.168.61.12
      access_ip: 192.168.61.12
    node4:
      ansible_host: 192.168.61.13
      ip: 192.168.61.13
      access_ip: 192.168.61.13
    node5:
      ansible_host: 192.168.61.14
      ip: 192.168.61.14
      access_ip: 192.168.61.14
    node6:
      ansible_host: 192.168.61.15
      ip: 192.168.61.15
      access_ip: 192.168.61.15
  children:
    kube_control_plane:
      hosts:
        node1:
        node2:
        node3:
    kube_node:
      hosts:
        node4:
        node5:
        node6:
    etcd:
      hosts:
        node1:
        node2:
        node3:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```

### Step 4: Run the Playbook

To start the Kubernetes installation, run the playbook using the ansible-playbook command:

```bash
ansible-playbook -i inventory/mycluster/inventory.ini --become --become-user=root cluster.yml
```

### Step 5: Access the Cluster

After the installation is complete, you can access your Kubernetes cluster. Copy the `kubeconfig` file from the control plane node to your local machine:

```bash
scp user@node1:~/.kube/config ~/.kube/config
```

Now you can access your cluster using `kubectl`:

```bash
kubectl get nodes
```
