# Kubernetes Deployment with Kubespray

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
git clone https://github.com/akbarmohammadi70/K8S_PostgresqlCluster_Nginx.git
cd 01-kubespray
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

## Setting Up MetalLB on Kubernetes


### Prerequisites

1. **An active Kubernetes cluster**: Ensure you have a running Kubernetes cluster. You can set one up using tools like `kubeadm`, `minikube`, or any cloud-based Kubernetes service.

2. **kubectl installed**: Ensure `kubectl` is installed and configured to communicate with your Kubernetes cluster.

### Installation Steps

#### 1. Install MetalLB

First, apply the MetalLB manifest to your cluster:

```sh
kubectl apply -f 02-metallb/metallb-native.yaml
```

This command will install MetalLB in the `metallb-system` namespace.

#### 2. Create a ConfigMap for MetalLB

MetalLB requires a ConfigMap to define the IP address pool that it can use. Create a file named `ippool.yaml` and `advertisment.yaml` with the following content:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.61.13-192.168.61.15
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
  nodeSelectors:
  - matchExpressions:
    - key: kubernetes.io/hostname
      operator: In
      values:
      - node4
      - node5
      - node6
  interfaces:
  - eth1

```

Apply the ConfigMap to your cluster:

```sh
kubectl apply -f 02-metallb/ippool.yaml
kubectl apply -f 02-metallb/advertisement.yaml
```

### Verifying the Installation

1. **Check MetalLB Pods**: Ensure that the MetalLB pods are running correctly.

```sh
kubectl get pods -n metallb-system
```

You should see output similar to:

```plaintext
NAME                          READY   STATUS    RESTARTS   AGE
controller-7b8b8b645f-lsz8v   1/1     Running   0          1m
speaker-9f9d6d9f5-lsh89       1/1     Running   0          1m
speaker-9f9d6d9f5-rj4cd       1/1     Running   0          1m
speaker-9f9d6d9f5-rj4cd       1/1     Running   0          1m
```

2. **Deploy a Service**: Create a Kubernetes service to verify that MetalLB assigns an external IP.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Deploy the service:

```sh
kubectl apply -f nginx-service.yaml
```

Check the service to see if an external IP is assigned:

```sh
kubectl get svc nginx-service
```

You should see an external IP address within the range you specified in the ConfigMap.

### Conclusion

You have successfully installed and configured MetalLB on your Kubernetes cluster. Your cluster can now utilize MetalLB to provide load balancing for services.

For more advanced configurations and troubleshooting, refer to the [MetalLB documentation](https://metallb.universe.tf/).

---

## Setting Up NGINX Ingress Controller on Kubernetes

### Prerequisites

1. **An active Kubernetes cluster**: Ensure you have a running Kubernetes cluster. You can set one up using tools like `kubeadm`, `minikube`, or any cloud-based Kubernetes service.

2. **kubectl installed**: Ensure `kubectl` is installed and configured to communicate with your Kubernetes cluster.

### Installation Steps

#### 1. Install the NGINX Ingress Controller

```sh
kubectl apply -f 03-nginx_controller/deploy.yaml
```

#### 2. Verify the Installation

Ensure that the NGINX Ingress Controller pods are running correctly:

```sh
kubectl get pods -n ingress-nginx
```

#### 3. Create an Ingress Resource

An Ingress resource routes traffic to your services. Create a file named `example-ingress.yaml` with the following content:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
```

Make sure to replace `example.com` with your actual domain and `example-service` with the name of the service you want to expose.

Apply the Ingress resource:

```sh
kubectl apply -f example-ingress.yaml
```

#### 5. Configure DNS

Ensure that your domain (e.g., `example.com`) points to the external IP of the NGINX Ingress Controller. You can find this IP by running:

```sh
kubectl get svc -n ingress-nginx
```

Look for the `EXTERNAL-IP` of the `ingress-nginx-controller` service.

### Conclusion

You have successfully installed and configured the NGINX Ingress Controller on your Kubernetes cluster. Your cluster can now route external traffic to your internal services using Ingress resources.

For more advanced configurations and troubleshooting, refer to the [NGINX Ingress Controller documentation](https://kubernetes.github.io/ingress-nginx/).

---

### PostgreSQL Cluster on Kubernetes with Helm

#### Prerequisites

- Kubernetes cluster with at least 3 nodes
- Helm installed and configured

### Installation Steps

#### 1. Clone the Repository

```sh
cd 04-postgresql_chart/postgresql-cluster
```

#### 2. Prepare the Values File

Edit the `values.yaml` file to configure your PostgreSQL cluster:

```yaml
replicaCount: 3

image:
  repository: postgres
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 5432

postgresql:
  username: postgres
  password: your_password
  database: your_database

persistence:
  enabled: true
  size: 5Gi
  storageClass: ""
  accessModes:
    - ReadWriteOnce
  hostPath: /mnt/data/postgresql

nodeSelector:
  kubernetes.io/hostname: ""

affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
            - key: "app"
              operator: In
              values:
                - postgresql
        topologyKey: "kubernetes.io/hostname"

backup:
  schedule: "0 2 * * *" # Daily at 2 AM
  storage:
    size: 5Gi
    hostPath: /mnt/data/postgresql-backups

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

#### 3. Create Kubernetes Secrets

Add the encoded password to .values.yaml for Kubernetes Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "postgresql-cluster.fullname" . }}
type: Opaque
data:
  postgresql-username: {{ .Values.postgresql.username | b64enc }}
  postgresql-password: {{ .Values.postgresql.password | b64enc }}

```

#### 4. Deploy the StatefulSet

Create the StatefulSet using Helm:

```sh
helm install postgresql-cluster ./postgresql-cluster
```

### Customizing the Deployment

#### Persistent Volume Configuration

Ensure you have the correct hostPath settings in `values.yaml`. For example:

```yaml
persistence:
  hostPath: /mnt/data/postgresql
```

#### Backup Configuration

Backups are scheduled using a cron job defined in `values.yaml`:

```yaml
backup:
  schedule: "0 2 * * *" # Adjust the schedule as needed
  storage:
    size: 5Gi
    hostPath: /mnt/data/postgresql-backups
```

### Monitoring and Management

Monitor the deployment using standard Kubernetes tools:

```sh
kubectl get pods
kubectl get svc
kubectl get pvc
```

### Uninstalling the Chart

To uninstall/delete the `postgresql-cluster` deployment:

```sh
helm uninstall postgresql-cluster
```

This command removes all the Kubernetes components associated with the chart and deletes the release.

### Conclusion

You have successfully deployed a PostgreSQL cluster on Kubernetes with Helm. The cluster includes 3 replicas with automatic failover, data persistence using hostPath, and scheduled backups. Customize the configuration as needed to fit your requirements.

---

# Nginx Deployment Helm Chart

## Prerequisites

- Kubernetes cluster
- Helm installed

## Installation

1. Clone this repository or navigate to the directory containing the Helm chart.

2. Install the Helm chart:

```bash
helm install nginx-release 05-nginx_chart/nginx-chart
```

This command will deploy the Nginx chart with the default configurations specified in `values.yaml`.

## Files

### Chart.yaml

Contains metadata about the chart, such as the name and version.

### values.yaml

Defines the default values for the chart. You can customize these values before deploying the chart.

```yaml
nginx:
  image:
    repository: nginx
    tag: 1.23.0
    pullPolicy: IfNotPresent
  replicas: 2

service:
  type: NodePort
  port: 80
  nodePort: 30080
```

## Usage

After the chart is installed, you can access Nginx via the NodePort. Assuming the node IP is `192.168.61.14`, you can make a request to Nginx:

```bash
curl http://192.168.61.14:30080/ -i
```

You should receive a response with the Pod IP in the body and in the `server_ip` header:

```
HTTP/1.1 200 OK
Server: nginx/1.23.0
Date: [current date]
Content-Type: text/plain
Content-Length: [length]
Connection: keep-alive
server_ip: [Pod IP]
[Pod IP]
```

This confirms that Nginx is correctly configured and running in your Kubernetes cluster.
