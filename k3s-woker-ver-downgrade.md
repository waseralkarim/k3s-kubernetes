**K3s Worker Node Version Downgrade Procedure**

When a worker node is running a newer k3s version than the control plane, it must be downgraded. Kubernetes requires that worker nodes never exceed the control plane version.

---

**Step 1: Identify the version mismatch**

```jsx
sudo kubectl get nodes -o wide
```

Note the control plane version and the mismatched worker node name/IP.

---

**Step 2: Get the node token from the master**

```jsx
sudo cat /var/lib/rancher/k3s/server/node-token
```

Save this token for the reinstall step.

---

**Step 3: Drain the worker node (from master)**

```jsx
sudo kubectl drain <NODE_NAME> --ignore-daemonsets --delete-emptydir-data --timeout=120s
```

This safely evicts all pods so they reschedule to other nodes.

---

**Step 4: Stop the agent and reinstall at the correct version (on the worker node)**

```jsx
sudo systemctl stop k3s-agent

curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION="<TARGET_VERSION>" \
  K3S_URL="https://<MASTER_IP>:6443" \
  K3S_TOKEN="<NODE_TOKEN>" \
  sh -
```

Replace the placeholders:

- `<TARGET_VERSION>` — the control plane version, e.g. `v1.30.5+k3s1`
- `<MASTER_IP>` — the control plane node's IP
- `<NODE_TOKEN>` — the token from Step 2

---

**Step 5: Verify the agent is running (on the worker node)**

```jsx
sudo systemctl status k3s-agent
```

Should show `active (running)`.

---

**Step 6: Uncordon the node (from master)**

```jsx
sudo kubectl uncordon <NODE_NAME>
```

---

**Step 7: Verify all nodes match**

```jsx
sudo kubectl get nodes -o wide
```

All nodes should show the same version and Status `Ready`.

---

**Rollback (if the node fails to rejoin)**

```jsx
# On the worker node — full uninstall
sudo /usr/local/bin/k3s-agent-uninstall.sh

# On master — remove the stale node object
sudo kubectl delete node <NODE_NAME>

# On the worker node — rejoin with the Step 4 install command
```

---

**Important notes:**

- Worker nodes can be up to 2 minor versions *behind* the control plane, but never *ahead*.
- The node token is at `/var/lib/rancher/k3s/server/node-token` on the master.
- Always drain before making version changes to avoid workload disruption.
- If the node hostname changes after reinstall, delete the old node object from the master manually.
