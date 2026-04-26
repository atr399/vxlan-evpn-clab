# Installation Guide

Complete setup guide to deploy this lab from scratch on a fresh Ubuntu 24.04 VM.

## Hardware Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| RAM | 24 GB | 32 GB |
| CPU | 6 cores | 8+ cores |
| Disk | 30 GB free | 50 GB free |

## Step 1 - Disable Hyper-V on Windows host

In PowerShell as Administrator:

```powershell
bcdedit /set hypervisorlaunchtype off
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName VirtualMachinePlatform -NoRestart
Disable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -NoRestart
```

Also disable Memory Integrity in Windows Security. Reboot Windows.

## Step 2 - VMware VM Settings

- Memory: 28-32 GB
- Processors: 8 vCPUs
- Check: Virtualize Intel VT-x/EPT
- Uncheck: Virtualize CPU performance counters

## Step 3 - Install base tools

```bash
sudo apt update
sudo apt install -y curl wget git ca-certificates gnupg lsb-release make build-essential
```

## Step 4 - Install Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

## Step 5 - Install KVM

```bash
sudo apt install -y qemu-kvm cpu-checker libvirt-daemon-system bridge-utils tcpdump
sudo usermod -aG kvm $USER
```

## Step 6 - Install Containerlab

```bash
bash -c "$(curl -sL https://get.containerlab.dev)"
```

## Step 7 - Reboot

```bash
sudo reboot
```

## Step 8 - Verify environment

```bash
docker --version
docker run hello-world
kvm-ok
containerlab version
groups
```

Expected: Docker runs hello-world without sudo, kvm-ok says "KVM acceleration can be used", groups include docker and kvm.

## Step 9 - Download Cisco NX-OSv image

Free CCO account required: https://id.cisco.com/signin/register

Download Nexus 9300v Lite from: https://software.cisco.com/download/home/286312239

Path: Switches > Data Center Switches > Nexus 9000 Series Switches > Nexus 9000v Switch > NX-OS System Software

This lab uses version 10.5(5) Lite, file nexus9300v64-lite.10.5.5.M.qcow2.

## Step 10 - Build vrnetlab Docker image

```bash
cd ~
git clone https://github.com/srl-labs/vrnetlab.git
cd vrnetlab/cisco/n9kv
mv ~/nexus9300v64-lite.10.5.5.M.qcow2 ./n9kv-9300-10.5.5.qcow2
make docker-image
```

Verify:

```bash
docker images | grep n9kv
```

## Step 11 - Clone this repo

```bash
mkdir -p ~/labs
cd ~/labs
git clone https://github.com/atr399/vxlan-evpn-clab.git vxlan-fabric
cd vxlan-fabric
```

## Step 12 - Deploy

```bash
sudo containerlab deploy -t vxlan-fabric.clab.yml
```

Wait 10-15 minutes for boot.

## Step 13 - Verify fabric

```bash
sudo containerlab inspect -t vxlan-fabric.clab.yml
ssh admin@<leaf1-ip>
```

Password is admin. In NX-OS:

```
show bgp l2vpn evpn summary
show nve peers
show nve vni
```

## Step 14 - Test data plane

```bash
docker exec -it clab-vxlan-fabric-host1 ping -c 5 192.168.10.2
```

## Step 15 - Capture VXLAN traffic (optional)

```bash
LEAF1_PID=$(docker inspect -f '{{.State.Pid}}' clab-vxlan-fabric-leaf1)
sudo nsenter -t $LEAF1_PID -n tcpdump -i eth1 -n udp port 4789 -vv
```

## Lifecycle

Stop the lab:

```bash
sudo containerlab destroy -t vxlan-fabric.clab.yml --cleanup
```

Start again:

```bash
sudo containerlab deploy -t vxlan-fabric.clab.yml
```

Save running configs after manual changes:

```bash
sudo containerlab save -t vxlan-fabric.clab.yml
for node in spine1 spine2 leaf1 leaf2; do
  sudo cp clab-vxlan-fabric/$node/config/startup-config.cfg saved-configs/${node}.cfg
done
sudo chown -R $USER:$USER saved-configs/
```

## Troubleshooting

**kvm-ok fails**: Hyper-V still active or VT-x not enabled in VMware. Recheck Steps 1-2.

**VMware VPMC error**: Uncheck "Virtualize CPU performance counters" in VM settings.

**SSH host key changed**: ssh-keygen -f ~/.ssh/known_hosts -R <ip>

**Saved configs lost after destroy**: The saved-configs/ directory survives --cleanup. Never store configs inside clab-vxlan-fabric/.

**NX-OS startup-config not fully applied**: NX-OS evaluates startup-config line by line. Commands referencing features that haven't been enabled yet are silently dropped. The saved-configs/ files were captured from a working show running-config and replay reliably.
