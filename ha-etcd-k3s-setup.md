## **Set your K3S_TOKEN**

Generate a token if you haven’t already:

```
export K3S_TOKEN=$(openssl rand -hex 16)
```

## **Install the first HA server (without Traefik)**

Run on **Server 1**:

```
curl -sfL https://get.k3s.io | \
K3S_TOKEN=$K3S_TOKEN sh -s - server \
  --cluster-init \
  --disable=traefik \
  --disable=servicelb
```

Explanation of flags:

| Flag | Meaning |
| --- | --- |
| `--cluster-init` | Creates a new HA embedded etcd cluster |
| `--disable=traefik` | Prevents Traefik ingress controller from being installed |
| `--tls-san` | Optional, specify fixed IP or hostname for API access |
|  `--disable=servicelb` | Disables default Klipper Loadbalancer |

## **Join additional HA servers (If you want multiple master)**

On **Server 2, Server 3, … etc.**

```
curl -sfL https://get.k3s.io | \
K3S_TOKEN=$K3S_TOKEN sh -s - server \
  --server https://<FIRST_SERVER_IP>:6443 \
  --disable=traefik 
```

Make sure to **repeat `--disable=traefik` on all servers**, otherwise Traefik will be installed on those nodes.

To make kubectl usable as a normal user (like ubuntu):

Run these commands:
```bash
mkdir -p $HOME/.kube
sudo cp /etc/rancher/k3s/k3s.yaml $HOME/.kube/config

export KUBECONFIG=$HOME/.kube/config

echo 'export KUBECONFIG=$HOME/.kube/config' >> ~/.bashrc
source ~/.bashrc

#Change ownership to devops
sudo chown ubuntu:ubuntu /home/ubuntu/.kube/config

# Fix permissions (readable by user)
chmod 600 /home/ubuntu/.kube/config
```

## **Verify cluster nodes**

```
kubectl get nodes
```

All servers should show as **Ready**.

## **Join worker nodes**

Get the **K3s token**:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
Get the internal IP of master node:

```bash
hostname -I
```

Workers can join the HA cluster normally:

```
curl -sfL https://get.k3s.io | K3S_TOKEN=<Token> sh -s - agent --server https://<ip-add/domain>:6443
```

No Traefik is installed on agents by default.


### **Tips**

- If you want to add **other ingress controllers later** (like NGINX or Contour), you can do that manually.
- Make sure **all servers use the same flags** (`-disable=traefik`, `-tls-san`, etc.) to avoid inconsistencies.
- HA embedded etcd **requires an odd number of server nodes** (3, 5, …).
