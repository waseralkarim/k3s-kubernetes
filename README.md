# K3s Lightweight Kubernetes

K3s is a highly available, certified Kubernetes distribution designed for production workloads in unattended, resource-constrained, remote locations or inside IoT appliances.

<img width="1182" height="681" alt="image" src="https://github.com/user-attachments/assets/9fe39896-0fb3-4d7a-83b7-3929a8db3580" />

## **Set up K3s Cluster**

### On the **Master Node**

```bash
curl -sfL https://get.k3s.io | sh -
```

### **Optional: Disable Traefik During K3s Install**

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik" sh -
```

This prevents Traefik from being installed, leaving you free to install NGINX ingress controller via Helm.

Get the **K3s token**:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
To make kubectl usable as a normal user (like devops):

Run these commands:

mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Get the internal IP of master node:

```bash
hostname -I
```

### On **Each Worker Node**

Replace `<MASTER_IP>` and `<NODE_TOKEN>` accordingly:

```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN="<NODE_TOKEN>" sh -
```

Verify on master node:

```bash
kubectl get nodes
```
