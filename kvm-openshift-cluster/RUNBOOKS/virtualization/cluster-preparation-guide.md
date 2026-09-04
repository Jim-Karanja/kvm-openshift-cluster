# System Preparation Guide: Swap, Kernel Modules, Containerd & Kubernetes

This comprehensive guide covers the essential host system preparation steps required for setting up container runtimes and Kubernetes nodes.

---

## Part 1: Disabling Swap Space

Kubernetes and certain container runtimes require swap space to be completely disabled to ensure predictable resource scheduling and performance stability.

### Step 1.1: Turn Off Swap Immediately
To deactivate all swap space on the running system immediately, execute:
```bash
sudo swapoff -a
This disables any active swap files or partitions for the current active session.
Step 1.2: Permanent Disable via /etc/fstab
To prevent swap from being re-enabled automatically upon system reboot, comment out any swap lines in /etc/fstab using this automated command:
sudo sed -i.bak '/swap/s/^/#/' /etc/fstab
This uses sed to search for lines containing "swap" in /etc/fstab, prepends a # to comment them out, and creates a backup of the original file as /etc/fstab.bak.
Step 1.3: Mask Systemd Swap Units
Ubuntu and other systemd-based distributions often utilize dynamic systemd swap or zram units. To prevent these services from automatically activating swap, run:
sudo systemctl mask \$(systemctl list-unit-files | grep -i swap | awk '{print \$1}') 2>/dev/null || true
This command lists all systemd unit files, filters for those containing "swap", extracts their names, and masks them so systemd cannot start them.
Step 1.4: Verify Swap is Deactivated
To confirm that swap is deactivated and no longer available on the system, run:
free -h | grep Swap
An output showing 0B or no swap lines confirms that swap has been successfully disabled.
Part 2: Configuring Kernel Modules
Container runtimes require specific Linux kernel modules to handle overlay filesystems and bridge networking.
Step 2.1: Load Required Kernel Modules
Load the overlay and br_netfilter modules into the active kernel:
sudo modprobe overlay
sudo modprobe br_netfilter
Why are these modules required?
overlay (OverlayFS): This driver allows the container runtime to layer container images (combining the read-only base image layers with the writeable container layer), which is highly efficient for container storage.
br_netfilter (Bridge Netfilter): This module enables packet filtering on bridged network interfaces. It allows the host's iptables to inspect and filter bridge network traffic, which is a fundamental requirement for Kubernetes networking and container network interfaces (CNIs).
Step 2.2: Verify Modules are Loaded
To confirm that both modules are successfully loaded into the kernel, run:
lsmod | grep -E "overlay|br_netfilter"
This command lists all currently loaded kernel modules and filters the output for overlay and br_netfilter. If loaded correctly, you will see entries for both in the terminal output.
Step 2.3: Persistent Module Loading (Optional but Recommended)
To ensure these modules load automatically on system reboot, add them to your modules-load.d configuration:
cat <<EOF | sudo tee /etc/modules-load.d/container-runtime.conf
overlay
br_netfilter
EOF
Part 3: Installing the Containerd Container Runtime
Containerd is an industry-standard, lightweight container runtime that manages the complete container lifecycle of its host system. Below are the steps to add the official repository and install the runtime.
Step 3.1: Update and Install Base Tools
Before adding new repositories, update the local package index and install the necessary package management helper tools:
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
apt-transport-https: Allows the package manager to securely fetch packages over HTTPS.
ca-certificates: Enables the verification of secure SSL/TLS connections.
curl: A tool to transfer data from or to a server, used here to download the repository's encryption key.
software-properties-common: Provides scripts for managing software repositories.*
Step 3.2: Configure Repository GPG Directory and Download Key
Create a dedicated directory to store trusted repository keys and fetch the official Docker/Containerd GPG key:
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
This downloads the ASCII armor-protected GPG key, de-armors it into a binary format, and saves it in /etc/apt/keyrings/ to verify the authenticity of the repository packages.
Step 3.3: Add Docker/Containerd APT Source
Inject the official repository source into your APT configurations:
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \$(lsb_release -cs) stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list
This command appends a formatted line defining the architecture (amd64), specifies the verification key, points to the official repository URL, and automatically resolves your Ubuntu distribution name (e.g., jammy for 22.04).
Step 3.4: Install the Containerd Runtime
With the repository configured, refresh your package indexes and install the containerd.io package:
sudo apt-get update
sudo apt-get install -y containerd.io
This pulls the package list from the newly added source and installs the pure containerd container engine, making it ready to run containers on your host.
Part 4: Configuring and Initializing Containerd
Once containerd is installed, you need to generate its default configuration and configure it to use the systemd cgroup driver. This is a critical step for Kubernetes node stability, as it prevents resource conflicts between systemd and the container runtime.
Step 4.1: Generate the Default Configuration
Create a configuration directory and generate the default configuration file for containerd:
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
This dumps the default, fully commented configuration schema into /etc/containerd/config.toml where you can modify specific operational settings.
Step 4.2: Enable the SystemdCgroup Driver
Configure containerd to use the host's systemd cgroup management slice. Modify the configuration inline using sed:
Why is this important? By default, containerd may use the cgroupfs driver, but if systemd is the init system on your host, you will end up with two distinct cgroup managers. Having both systems try to monitor and allocate system resources (CPU, Memory) can cause resource starvation and system instability under high load. Switching this to true ensures that containerd delegates resource management to systemd.*
Step 4.3: Initialize, Enable, and Verify the Service
Apply the configurations by restarting the daemon, configuring it to start automatically on system boot, and checking its operational health:
sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd
Ensure that the status command output shows active (running). If there are syntax errors in your /etc/containerd/config.toml, containerd will fail to start and display details in this status or in your system journal (journalctl -u containerd).
Part 5: Installing Kubernetes Tools (kubeadm, kubelet, and kubectl)
This section details how to configure the community-owned package repositories to install the core Kubernetes node components.
Step 5.1: Elevate to Root
To carry out package and repository setups directly without prepending sudo to every command, start by elevating your shell to a root session:
sudo -i
This places you in an interactive administrative shell. The subsequent commands in this section assume you are running as the root user.
Step 5.2: Configure Kubernetes Package Repository and Key
Kubernetes packages are signed to verify authenticity. Download the official GPG key and add the Kubernetes v1.29 community repository:
# Create directory for apt keyrings if it doesn't exist
mkdir -p /etc/apt/keyrings

# Download the v1.29 Release GPG key and de-armor it
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the official Kubernetes deb repository source to your sources list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
  | tee /etc/apt/sources.list.d/kubernetes.list
gpg --dearmor: Converts the GPG key from a text-based ASCII format to a binary format, saving it securely in your keyrings folder.
pkgs.k8s.io: This is the official, community-managed repository path for stable release packages of Kubernetes.*
Step 5.3: Install Kubernetes Cluster Components
Refresh your system packages and install the three primary tools required on every Kubernetes cluster node:
# Update the APT package cache
apt-get update

# Install the packages
apt-get install -y kubeadm kubelet kubectl

# Mark packages to prevent automated updates
apt-mark hold kubeadm kubelet kubectl
What do these packages do?
kubeadm: The command-line utility used to bootstrap, configure, and upgrade the Kubernetes cluster components.
kubelet: The crucial node agent that runs on all hosts in the cluster. It receives commands from the Control Plane API server and is responsible for making sure containers are running properly inside Pods on its local system.
kubectl: The client CLI used to issue commands to the cluster control plane for managing services, workloads, and inspecting cluster-wide state.
apt-mark hold: This tells the system package manager to "lock" or "hold" the installed version of these packages. This is a critical best practice to prevent automatic background upgrades from introducing breaking changes or version mismatches within your cluster nodes.*
Step 5.4: Enable and Initialize the Kubelet Service
With the binaries installed, enable the kubelet system service so that it automatically runs on boot:
systemctl enable --now kubelet
