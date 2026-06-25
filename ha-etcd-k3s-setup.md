## **Set your K3S_TOKEN**

### Disable swap first
```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```
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
### Get k3s token
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
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

## Useful commands

List leases in a specific namespace:


```
kubectl get lease -n <namespace-name>
```


List node heartbeats (Node Leases):

```
kubectl get lease -n kube-node-lease
```

k3s doesn't ship `etcdctl`, so install it once:


```bash
sudo apt-get install -y etcd-client
```

Set the cert flags once so you don't repeat them (k3s puts them under its tls dir):

```bash
export ETCDCTL_API=3
export ETCDCTL_CACERT=/var/lib/rancher/k3s/server/tls/etcd/server-ca.crt
export ETCDCTL_CERT=/var/lib/rancher/k3s/server/tls/etcd/server-client.crt
export ETCDCTL_KEY=/var/lib/rancher/k3s/server/tls/etcd/server-client.key
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
```

(These files are root-only, so run the `etcdctl` commands with `sudo -E` to carry the env, or just `sudo` and pass `--cacert/--cert/--key` explicitly.)

**Member → node mapping:**



```bash
sudo -E etcdctl member list -w table
```

The `NAME` column is the node hostname and `PEER ADDRS` is that node's IP (`:2380`) — that's exactly "which etcd member maps to which master."

**Which member is the leader (and DB health):**



```bash
sudo -E etcdctl endpoint status --cluster -w table
```

`--cluster` auto-discovers all members, so you get one row per master. The `IS LEADER` column shows `true` on exactly one — that's the current Raft leader. The same table gives you DB size and Raft term/index.

**Quick health check across all members:**



```bash
sudo -E etcdctl endpoint health --cluster -w table
```

### **Tips**

- If you want to add **other ingress controllers later** (like NGINX or Contour), you can do that manually.
- Make sure **all servers use the same flags** (`-disable=traefik`, `-tls-san`, etc.) to avoid inconsistencies.
- HA embedded etcd **requires an odd number of server nodes** (3, 5, …).
