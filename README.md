# Cisco VXLAN-EVPN Fabric Lab

A reproducible Cisco data center fabric simulation using [Containerlab](https://containerlab.dev/) and Cisco Nexus 9000v Lite (NX-OS 10.5.5).

Demonstrates leaf-spine architecture with iBGP EVPN control plane, VXLAN data plane, ECMP underlay, and route reflection — running entirely in Docker on a single Linux host.

## Topology

```
                ┌──────────┐         ┌──────────┐
                │  spine1  │         │  spine2  │
                │  AS65000 │         │  AS65000 │
                │   (RR)   │         │   (RR)   │
                └──┬────┬──┘         └──┬────┬──┘
                   │    └─────┐    ┌────┘    │
                   │          │    │         │
                ┌──┴──────────┴┐ ┌─┴─────────┴┐
                │    leaf1     │ │   leaf2    │
                │ Lo0:10.0.0.1 │ │Lo0:10.0.0.2│
                │   AS65000    │ │  AS65000   │
                └──────┬───────┘ └─────┬──────┘
                       │               │
                   ┌───┴────┐      ┌───┴────┐
                   │ host1  │      │ host2  │
                   │.10.1/24│      │.10.2/24│
                   └────────┘      └────────┘
                  VLAN 10  /  VNI 10010  /  192.168.10.0/24
```

## Architecture

| Layer | Protocol | Role |
|---|---|---|
| Underlay | OSPF area 0 | Loopback reachability, ECMP across spines |
| Overlay control plane | iBGP L2VPN-EVPN (AS 65000) | MAC/VTEP advertisement |
| Overlay data plane | VXLAN (UDP 4789) | Tunneled tenant traffic |
| Route reflection | Both spines act as RRs | Scalable iBGP without full mesh |
| BUM handling | Ingress replication | Per-VTEP unicast copies |

## Quick Start

For environments already set up with Docker, KVM, Containerlab, and the `vrnetlab/cisco_n9kv:9300-10.5.5` image:

```bash
git clone https://github.com/atr399/vxlan-evpn-clab.git
cd vxlan-evpn-clab
sudo containerlab deploy -t vxlan-fabric.clab.yml
```

Wait ~15 minutes for boot, then verify:

```bash
sudo containerlab inspect -t vxlan-fabric.clab.yml
docker exec -it clab-vxlan-fabric-host1 ping 192.168.10.2
```

**Setting up from scratch?** See [INSTALL.md](INSTALL.md) for the complete guide.

## Verification Commands

SSH to any leaf (`ssh admin@<leaf-ip>`, password `admin`):

```
show ip ospf neighbors          # underlay
show bgp l2vpn evpn summary     # overlay control plane
show nve peers                  # VTEP discovery
show nve vni                    # active VNIs
show l2route evpn mac all       # learned MACs via EVPN
```

Capture VXLAN encapsulation on the wire:

```bash
LEAF1_PID=$(docker inspect -f '{{.State.Pid}}' clab-vxlan-fabric-leaf1)
sudo nsenter -t $LEAF1_PID -n tcpdump -i eth1 -n udp port 4789 -vv
```

## Resource Requirements

| Component | RAM | vCPU |
|---|---|---|
| Each n9kv-lite node | 6 GB | 2 |
| Total fabric (4 NX-OS + 2 hosts) | ~25 GB | 8 |

## Repo Layout

```
.
├── README.md                   This file
├── INSTALL.md                  Full setup guide from a fresh Ubuntu VM
├── vxlan-fabric.clab.yml       Containerlab topology
├── saved-configs/              Source-of-truth NX-OS configs (used at boot)
└── configs/                    Original config templates
```

## License

MIT. Use freely for learning and labs.

## Author

Built by [@atr399](https://github.com/atr399).
