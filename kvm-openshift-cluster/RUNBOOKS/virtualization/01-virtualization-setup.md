# KVM Ubuntu Cluster & Host Setup Guide
## 1. Architecture Overview

### Target VM Configuration

| Node Name | vCPUs | RAM | Disk Size | IP Address | MAC Address | Hostname / FQDN |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`master`** | 4 | 4 GB | 50 GB | `<MASTER_IP>` | `<MASTER_MAC>` | `master.<COMPANY_DOMAIN>` |
| **`node1`** | 2 | 2 GB | 50 GB | `<NODE1_IP>` | `<NODE1_MAC>` | `node1.<COMPANY_DOMAIN>` |
| **`node2`** | 2 | 2 GB | 50 GB | `<NODE2_IP>` | `<NODE2_MAC>` | `node2.<COMPANY_DOMAIN>` |

### Host Storage Configuration
* **Drive**: Unused secondary disk on RHEL host (e.g., `/dev/sda`).
* **Target Volume Group**: `rhel` (or actual VG name from `vgs`).
* **Target Logical Volume**: `/dev/rhel/var` (Expanded to accommodate VM disk images).

---

## 2. Phase 1: Initial Host Access & Termius Setup

### 1. Discover Host IP Address
Run this command on the local console of the RHEL host:
```bash
# Display network interfaces and identify the primary IPv4 address
ip a s
```

### 2. Connect via Termius
1. Launch **Termius** on your workstation.
2. Add a new Host entry with the host IP (`<HOST_IP>`) and user credentials.
3. Establish an SSH session:
   ```bash
   ssh <USERNAME>@<HOST_IP>
   ```

---

## 3. Phase 2: Red Hat Subscription & System Registration

Register the RHEL host to enable repository access and package installation.

```bash
# 1. Register the system using Red Hat Subscription Manager
sudo subscription-manager register --username <REDHAT_USERNAME> --password <REDHAT_PASSWORD>

# 2. Attach available subscriptions automatically
sudo subscription-manager attach --auto

# 3. Verify active subscriptions
sudo subscription-manager list --consumed

# 4. Enable required RHEL repositories
sudo subscription-manager repos --enable=rhel-8-for-x86_64-baseos-rpms \
                                --enable=rhel-8-for-x86_64-appstream-rpms
```

---

## 4. Phase 3: KVM Virtualization Stack & Cockpit Installation

Install KVM hypervisor tools, Libvirt, and the Cockpit web interface.

```bash
# 1. Install virtualization modules and management tools
sudo dnf module install -y virt
sudo dnf install -y virt-install virt-viewer libvirt-devel cockpit cockpit-machines

# 2. Enable and start virtualization daemon and Cockpit socket
sudo systemctl enable --now libvirtd
sudo systemctl enable --now cockpit.socket

# 3. Allow Cockpit and Libvirt through the host firewall
sudo firewall-cmd --add-service=cockpit --permanent
sudo firewall-cmd --add-service=libvirt --permanent
sudo firewall-cmd --reload
```

* **Cockpit Web UI Access**: `https://<HOST_IP>:9090`

---

## 5. Phase 4: Host Storage Expansion (LVM)

Expand the host `/var` directory to ensure adequate space for VM images in `/var/lib/libvirt/images`.

```bash
# 1. List available storage devices and verify target disk (e.g., /dev/sda)
sudo lsblk

# 2. Create an LVM Physical Volume on the target device
sudo pvcreate /dev/sda

# 3. Extend the Volume Group (replace 'rhel' with your actual Volume Group name)
sudo vgextend rhel /dev/sda

# 4. Extend the Logical Volume for /var (e.g., +300GB)
sudo lvextend -L +300G /dev/rhel/var

# 5. Grow the XFS filesystem
sudo xfs_growfs /var
```

---

## 6. Phase 5: Static IP Reservation via Libvirt DHCP

Bind static IP addresses to virtual machine MAC addresses on the libvirt `default` network.

```bash
# 1. Add DHCP host entries to the default virtual network
sudo virsh net-update default add-last ip-dhcp-host \
  '<host mac="<MASTER_MAC>" name="master" ip="<MASTER_IP>"/>' --live --config

sudo virsh net-update default add-last ip-dhcp-host \
  '<host mac="<NODE1_MAC>" name="node1" ip="<NODE1_IP>"/>' --live --config

sudo virsh net-update default add-last ip-dhcp-host \
  '<host mac="<NODE2_MAC>" name="node2" ip="<NODE2_IP>"/>' --live --config

# 2. Restart the libvirt default network to commit changes
sudo virsh net-destroy default
sudo systemctl restart libvirtd
sudo virsh net-start default
```

---

## 7. Phase 6: VM Provisioning & OS Installation

Create and launch the virtual machines using `virt-install`.

```bash
# Provision Master Node
sudo virt-install \
  --name master \
  --ram 4096 \
  --vcpus 4 \
  --disk path=/var/lib/libvirt/images/master.qcow2,size=50 \
  --mac <MASTER_MAC> \
  --network network=default \
  --cdrom /var/lib/libvirt/images/ubuntu-20.04.6-live-server-amd64.iso \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole

# Provision Worker Node 1
sudo virt-install \
  --name node1 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/node1.qcow2,size=50 \
  --mac <NODE1_MAC> \
  --network network=default \
  --cdrom /var/lib/libvirt/images/ubuntu-20.04.6-live-server-amd64.iso \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole

# Provision Worker Node 2
sudo virt-install \
  --name node2 \
  --ram 2048 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/node2.qcow2,size=50 \
  --mac <NODE2_MAC> \
  --network network=default \
  --cdrom /var/lib/libvirt/images/ubuntu-20.04.6-live-server-amd64.iso \
  --graphics vnc,listen=0.0.0.0 \
  --noautoconsole
```

### OS Setup Checklist (via Cockpit Console)
1. Open Cockpit (`https://<HOST_IP>:9090`) $\rightarrow$ **Virtual Machines** $\rightarrow$ Select VM $\rightarrow$ **Console**.
2. **Language/Keyboard**: English.
3. **Partitioning**: Entire disk (`/dev/vda`).
4. **Hostname**: Set matching FQDN (`master.<COMPANY_DOMAIN>`, `node1.<COMPANY_DOMAIN>`, `node2.<COMPANY_DOMAIN>`).
5. **User**: `ubuntu`.
6. **OpenSSH Server**: Check **`[X] Install OpenSSH Server`** (Required).
7. Complete installation until **Install complete!** appears.

---

## 8. Phase 7: Post-Install Setup & ISO Detachment

Detach the ISO installation media to avoid rebooting into the installer.

```bash
# 1. Detach ISO virtual CD-ROM drive
sudo virsh detach-disk master sda --config 2>/dev/null || sudo virsh detach-disk master sdb --config
sudo virsh detach-disk node1 sda --config 2>/dev/null || sudo virsh detach-disk node1 sdb --config
sudo virsh detach-disk node2 sda --config 2>/dev/null || sudo virsh detach-disk node2 sdb --config

# 2. Restart virtual machines to boot from hard disk
sudo virsh destroy master && sudo virsh start master
sudo virsh destroy node1 && sudo virsh start node1
sudo virsh destroy node2 && sudo virsh start node2

# 3. Verify active DHCP leases
sudo virsh net-dhcp-leases default
```

---

## 9. Phase 8: Passwordless SSH Key Configuration

Configure passwordless SSH authentication from the host to each VM.

```bash
# 1. Generate SSH key pair on the RHEL host
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# 2. Copy the public key to each cluster node
ssh-copy-id ubuntu@<MASTER_IP>
ssh-copy-id ubuntu@<NODE1_IP>
ssh-copy-id ubuntu@<NODE2_IP>

# 3. Test passwordless connection
ssh ubuntu@<MASTER_IP>
```
