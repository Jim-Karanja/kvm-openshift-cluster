# System Preparation Guide: Swap, Kernel Modules, Containerd & Kubernetes

This guide covers the essential host system preparation steps required for setting up container runtimes and Kubernetes v1.29 nodes on Ubuntu.

---

## 1. Disabling Swap Space

Kubernetes requires swap space to be completely disabled to ensure predictable resource scheduling, proper container memory limits, and node stability.

### 1.1 Turn Off Swap Immediately
Deactivate all swap space on the running system:

```bash
sudo swapoff -a
```

### 1.2 Permanently Disable Swap via `/etc/fstab`
Prevent swap from re-enabling upon reboot by commenting out any swap entries:

```bash
sudo sed -i.bak '/swap/s/^/#/' /etc/fstab
```

### 1.3 Mask Systemd Swap Units
Prevent systemd dynamic swap or zram units from activating:

```bash
sudo systemctl mask $(systemctl list-unit-files | grep -i swap | awk '{print $1}') 2>/dev/null || true
```

### 1.4 Verify Swap Deactivation
Confirm swap is fully deactivated (must show `0B`):

```bash
free -h | grep Swap
```

---

## 2. Configuring Kernel Modules & Sysctl

Container runtimes require specific Linux kernel modules to handle overlay filesystems and bridge networking rules.

### 2.1 Persist Kernel Modules Across Reboots
Configure the modules to load automatically on boot:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

### 2.2 Load Modules into Running Kernel
Activate the modules immediately in the running session:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

> **Module Explanations:**
> * **`overlay` (OverlayFS):** Allows the container runtime to layer container images (combining read-only base layers with writeable container layers) efficiently.
> * **`br_netfilter` (Bridge Netfilter):** Enables packet filtering on bridged network interfaces, allowing host `iptables` to inspect and filter bridge network traffic (required for CNI plugins).

### 2.3 Apply Network Sysctl Parameters
Configure IP forwarding and bridge netfilter parameters for Kubernetes networking:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 2.4 Verify Loaded Modules
Confirm that both modules are active:

```bash
lsmod | grep -E "overlay|br_netfilter"
```

---

## 3. Installing Containerd Runtime

`containerd` manages the complete container lifecycle of its host system.

### 3.1 Install Base Dependencies
Update package indexes and install necessary helper tools:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
```

### 3.2 Add Docker/Containerd Repository GPG Key
Download and save the official GPG key:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

### 3.3 Add APT Repository Source
Add the official Docker/Containerd repository:

```bash
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list
```

### 3.4 Install Containerd
Refresh package lists and install `containerd.io`:

```bash
sudo apt-get update
sudo apt-get install -y containerd.io
```

---

## 4. Configuring Containerd

Configure `containerd` to enable the CRI plugin and utilize `systemd` for cgroup management.

### 4.1 Generate Default Configuration
Create a default configuration file:

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

### 4.2 Enable SystemdCgroup Driver
Update the configuration to delegate cgroup management to `systemd`:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml
```

> **Why this matters:** `kubelet` defaults to the `systemd` cgroup driver on Ubuntu. Matching this setting in `containerd` prevents dual-cgroup driver conflicts between `systemd` and `cgroupfs`, which cause node instability and failed cluster joins.

### 4.3 Enable and Start Containerd Service
Restart and enable the service:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
```

---

## 5. Installing Kubernetes Tools (`kubeadm`, `kubelet`, `kubectl`)

Install the Kubernetes core binaries pinned to version 1.29.

### 5.1 Elevate to Root Shell
Elevate privileges to streamline installation:

```bash
sudo -i
```

### 5.2 Add Kubernetes Package Repository
Download the v1.29 GPG key and add the single-line repository entry:

```bash
mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
  > /etc/apt/sources.list.d/kubernetes.list
```

### 5.3 Install & Version-Lock Kubernetes Binaries
Install the packages and set an `apt-mark hold` to block unintended updates:

```bash
apt-get update
apt-get install -y kubeadm kubelet kubectl
apt-mark hold kubeadm kubelet kubectl
```

| Tool | Purpose |
|---|---|
| **`kubeadm`** | CLI tool used to bootstrap, configure, and join Kubernetes nodes. |
| **`kubelet`** | Primary node agent that receives PodSpec instructions and ensures containers run properly. |
| **`kubectl`** | Administrative CLI used to interact with the Kubernetes API server. |
| **`apt-mark hold`** | Locks package versions to prevent automatic background upgrades from breaking cluster state. |

### 5.4 Enable Kubelet Service
Enable `kubelet` to ensure it boots automatically:

```bash
systemctl enable --now kubelet
```

> **Note:** The `kubelet` service will crash-loop in `activating` state until `kubeadm init` or `kubeadm join` is executed in the next stage. This is expected behavior.
