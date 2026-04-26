# Installation Guide

Complete setup guide to deploy this lab from scratch on a fresh Ubuntu 24.04 VM.

## Hardware Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| RAM | 24 GB | 32 GB |
| CPU | 6 cores | 8+ cores |
| Disk | 30 GB free | 50 GB free |
| CPU virtualization | VT-x/AMD-V required | with EPT/RVI |

## Hypervisor Setup (VMware Workstation Example)

This lab runs Cisco NX-OSv inside KVM inside Ubuntu inside VMware Workstation. Nested virtualization must be working for KVM acceleration.

### Step 1 — Disable Hyper-V on Windows host

If using VMware Workstation on Windows, Hyper-V must be disabled or it will block VT-x passthrough to nested VMs.

Check current state in PowerShell:
\`\`\`powershell
systeminfo | findstr /i "hyper-v"
\`\`\`

If you see `"A hypervisor has been detected"`, disable Hyper-V (PowerShell as Administrator):
\`\`\`powershell
bcdedit /set hypervisorlaunchtype off
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -NoRestart
\`\`\`

Also disable **Memory Integrity** in Windows Security → Device Security → Core isolation → toggle OFF.

Reboot Windows.

### Step 2 — Configure VMware VM Settings

In VMware Workstation, edit the Ubuntu VM:
- **Memory**: 28-32 GB
- **Processors**: 8 vCPUs
- **Processors → Virtualization engine**:
  - ✅ Virtualize Intel VT-x/EPT or AMD-V/RVI
  - ❌ Virtualize CPU performance counters (disable to avoid VPMC errors)
- **Disk**: at least 50 GB

## Ubuntu VM Setup

### Step 3 — Update and install base tools

\`\`\`bash
sudo apt update
sudo apt install -y curl wget git ca-certificates gnupg lsb-release make build-essential
\`\`\`

### Step 4 — Install Docker

\`\`\`bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
\`\`\`

### Step 5 — Install KVM and tools

\`\`\`bash
sudo apt install -y qemu-kvm cpu-checker libvirt-daemon-system bridge-utils tcpdump
sudo usermod -aG kvm $USER
\`\`\`

### Step 6 — Install Containerlab

\`\`\`bash
bash -c "$(curl -sL https://get.containerlab.dev)"
\`\`\`

### Step 7 — Reboot to apply group changes

\`\`\`bash
sudo reboot
\`\`\`

### Step 8 — Verify everything

After reboot:

\`\`\`bash
docker --version              # Docker 27.x or newer
docker run hello-world        # must succeed without sudo
kvm-ok                        # must say "KVM acceleration can be used"
containerlab version          # 0.74.x or newer
groups                        # must include docker and kvm
ls /dev/kvm                   # must exist
\`\`\`

If `kvm-ok` fails, recheck Steps 1-2 (Hyper-V on host, VMware VT-x checkbox).

## Nexus 9000v Image Setup

### Step 9 — Download the Cisco NX-OSv qcow2

Cisco requires a free CCO account. Sign up at https://id.cisco.com/signin/register if needed.

Download the **Nexus 9300v Lite** image:
- Portal: https://software.cisco.com/download/home/286312239
- Path: Switches → Data Center Switches → Nexus 9000 Series Switches → Nexus 9000v Switch → NX-OS System Software
- Version used in this lab: **10.5(5)** Lite
- File: `nexus9300v64-lite.10.5.5.M.qcow2` (~1.5 GB)

Transfer the qcow2 to your Ubuntu VM (via VMware Shared Folders, scp, or HTTP).

### Step 10 — Build the vrnetlab Docker image

\`\`\`bash
cd ~
git clone https://github.com/srl-labs/vrnetlab.git
cd vrnetlab/cisco/n9kv

# Move and rename qcow2 to match the Makefile expected pattern
mv ~/nexus9300v64-lite.10.5.5.M.qcow2 ./n9kv-9300-10.5.5.qcow2

# Build (~5-10 minutes)
make docker-image
\`\`\`

Verify the image was built:
\`\`\`bash
docker images | grep n9kv
# Expected: vrnetlab/cisco_n9kv   9300-10.5.5   ...   ~2.5GB
\`\`\`

If you use a different NX-OS version, update the `image:` field in `vxlan-fabric.clab.yml` to match the tag.

## Lab Deployment

### Step 11 — Clone this repo

\`\`\`bash
cd ~/labs
git clone https://github.com/atr399/vxlan-evpn-clab.git vxlan-fabric
cd vxlan-fabric
\`\`\`

### Step 12 — Deploy the fabric

\`\`\`bash
sudo containerlab deploy -t vxlan-fabric.clab.yml
\`\`\`

Allow **10-15 minutes** for all 4 NX-OS nodes to fully boot. Monitor with:
\`\`\`bash
docker ps                 # all 6 containers running
free -h                   # ~25 GB used during boot
\`\`\`

### Step 13 — Verify

\`\`\`bash
sudo containerlab inspect -t vxlan-fabric.clab.yml
\`\`\`

Note the assigned IP addresses (they may differ each deploy). SSH to leaf1:
\`\`\`bash
ssh admin@<leaf1-ip>      # password: admin
\`\`\`

Check the fabric:
\`\`\`
show bgp l2vpn evpn summary
show nve peers
show nve vni
\`\`\`

Expected:
- 2 BGP neighbors (both spines), state **Established**
- 1 NVE peer (the other leaf), state **Up**
- VNI 10010 active, mode CP (Control Plane)

### Step 14 — Test data plane

\`\`\`bash
docker exec -it clab-vxlan-fabric-host1 ping -c 5 192.168.10.2
\`\`\`

Expected: 5 replies, 0% packet loss. First ping ~15-20 ms (ARP + EVPN learning), rest single-digit.

### Step 15 — Capture VXLAN encapsulation (optional)

\`\`\`bash
LEAF1_PID=$(docker inspect -f '{{.State.Pid}}' clab-vxlan-fabric-leaf1)
sudo nsenter -t $LEAF1_PID -n tcpdump -i eth1 -n udp port 4789 -vv
\`\`\`

In another terminal, ping host2 from host1. The tcpdump will show VXLAN-encapsulated frames on UDP/4789 with VNI 10010.

## Lifecycle Operations

### Stop the lab (frees RAM, keeps configs)
\`\`\`bash
sudo containerlab destroy -t vxlan-fabric.clab.yml --cleanup
\`\`\`

### Restart the lab
\`\`\`bash
sudo containerlab deploy -t vxlan-fabric.clab.yml
\`\`\`

### Save running configs after manual changes
\`\`\`bash
sudo containerlab save -t vxlan-fabric.clab.yml

# Copy to safe location (saved-configs/ survives --cleanup)
for node in spine1 spine2 leaf1 leaf2; do
  sudo cp clab-vxlan-fabric/$node/config/startup-config.cfg saved-configs/${node}.cfg
done
sudo chown -R $USER:$USER saved-configs/
\`\`\`

## Troubleshooting

### `kvm-ok` says KVM acceleration can NOT be used
Hyper-V is still active on the Windows host, or VT-x/EPT is not enabled in VMware VM settings. Recheck Steps 1-2.

### VMware VPMC error on VM boot
Disable "Virtualize CPU performance counters" in VMware VM Settings → Processors.

### NX-OS startup-config not fully applied
NX-OS evaluates startup-config line by line. Commands referencing features that haven't been enabled yet are silently dropped. The provided `saved-configs/` files were captured from a working `show running-config` and replay reliably. If you regenerate configs, ensure `feature nv overlay` and `nv overlay evpn` appear before any `router bgp ... address-family l2vpn evpn` blocks.

### `--cleanup` deleted my saved configs
The `clab-vxlan-fabric/` directory is fully wiped by `--cleanup`. Always store source-of-truth configs in `saved-configs/` (outside `clab-vxlan-fabric/`).

### SSH "REMOTE HOST IDENTIFICATION HAS CHANGED"
Containerlab reuses IPs across deploys. Clear the stale key:
\`\`\`bash
ssh-keygen -f ~/.ssh/known_hosts -R <ip>
\`\`\`

Or add to `~/.ssh/config`:
\`\`\`
Host 172.20.20.*
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
    LogLevel ERROR
\`\`\`

## References

- Containerlab docs: https://containerlab.dev/
- Cisco Nexus 9000v guide: https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/n9000v-9300v-9500v.html
- srl-labs/vrnetlab: https://github.com/srl-labs/vrnetlab
- Cisco VXLAN-EVPN Configuration Guide: https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/93x/vxlan/configuration/guide/b-cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-93x.html
