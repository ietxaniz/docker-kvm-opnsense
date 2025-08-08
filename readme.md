# Multi-Segment VPN Homelab with Debian, KVM, and OPNsense

This project documents a modular, real-world homelab architecture built on Linux. It combines KVM virtual machines, host-exposed Docker services, OPNsense firewall/routing — all tied together through segmented virtual LANs with various VPN configurations.

![](./figs/architecture.drawio.svg)

## 🔧 Stack Overview

- **Host OS**: Debian 13 with KDE *(tested; adaptable to others)*
- **Virtualization**: KVM / QEMU + libvirt
- **Routing & Firewall**: OPNsense VM with multiple virtual interfaces
- **Containerization**: Docker on host

---

## 🖥️ Tested Hardware

- **CPU**: AMD 8700G *(host runs on iGPU)*
- **RAM**: 128 GB *(32 GB likely sufficient)*
- **GPU**: NVIDIA 4060 Ti

---

## 📚 Contents

| Section | Description |
|--------|-------------|
| [📐 Architecture Overview](docs/architecture.md) | LAN design, VPN routing paths, interface roles |
| [🌐 Network Setup](docs/network-setup.md) | Creating bridges, IP ranges, NIC config |
| [🧱 Installing KVM and OPNsense](docs/tools-installation.md) | Base VM setup, WAN/LAN mapping |
| [📁 File Sharing & Host Services](docs/file-sharing-and-host-services.md) | Expose containers to segmented LANs |
| [🔌 LAN10](docs/lan10.md) | Direct internet access |
| [🔌 LAN20](docs/lan20.md) | VPN with port forwarding *(WireGuard)* |
| [🔌 LAN30](docs/lan30.md) | VPN with switchable IPs *(WireGuard pools)* |
| [🔌 LAN40](docs/lan40.md) | Host services network *(privileged access)* |
| [🔌 LAN50](docs/lan50.md) | No internet access *(isolated workloads)* |
| [🔌 LAN99](docs/lan99.md) | Management network security |
| [⚙️ VM Startup Orchestration](docs/vm-startup.md) | Delay VMs until OPNsense is fully online |
| [🛠️ Troubleshooting](docs/troubleshooting.md) | iGPU VRAM, bridge issues, network quirks |

---

## 📁 File Sharing & Host Services

This setup solves the common problem of sharing files and host-based services with virtual machines — not by using VirtualBox-style folder mounts, but by exposing Docker containers as proper network services, routed and secured through OPNsense.

> 💡 Example: Run a Samba container that shares a host folder, and access it from any VM as if it were a NAS. OPNsense can control which VMs see it, log access, and isolate or expose it as needed — just like with real infrastructure.

All containers operate in a dedicated IP space (e.g. `172.16.0.0/12`) and are reachable from VMs via static routes and firewall rules defined in OPNsense.

---

## 🔀 LAN Segment Purposes

Each LAN is isolated and routed via OPNsense. These reflect **real-world network scenarios** with different trust levels and access patterns:

| Segment | Purpose                    | VPN Type   | Example Use Cases               |
|---------|----------------------------|------------|---------------------------------|
| LAN10   | Direct internet            | None       | Gaming, streaming, general use  |
| LAN20   | VPN with port forwarding   | WireGuard  | P2P, servers, incoming connections |
| LAN30   | Switchable VPN IPs         | WireGuard  | Geographic routing, IP rotation |
| LAN40   | Host services network      | None       | CI/CD, development, administration |
| LAN50   | No internet access         | None       | Secure dev, testing, internal   |
| LAN99   | Management network         | None       | OPNsense access, infrastructure |

> 💡 VMs can be switched between LANs like moving an Ethernet cable — just assign them to a different KVM bridge.

---

## 🧠 Philosophy

This guide is meant for users comfortable with Linux. Not every detail is explained step-by-step — but if you get stuck, a quick search or help from an LLM will usually get you unstuck.

Core principles:

- **Modularity**: Each LAN or VM has a clear, isolated purpose
- **Reproducibility**: Most of it works across distros and setups — not just Debian/KDE
- **Realism**: Everything here has been tested on real hardware with real VPN providers

---

## 🧪 Tested VPN Providers

This setup supports different classes of VPN services:

- **WireGuard with port forwarding**: [OVPN.com](https://www.ovpn.com)
- **WireGuard with IP rotation**: Custom configs with multi-endpoint pools ([OVPN.com](https://www.ovpn.com))

> 💡 Even without a VPN, you can still use LAN10 (direct), LAN40 (host services), or LAN50 (isolated) for various workloads.

---

## 📎 Contributions

This is an opinionated setup based on real-world experience.  
Suggestions, fixes, or improvements are welcome — feel free to:

- Open issues
- Fork and adapt
- Submit a PR

---

## 📥 Get Started

Clone the repo and begin with the [Architecture Overview](docs/architecture.md):

```bash
git clone https://github.com/ietxaniz/docker-kvm-opnsense
cd docker-kvm-opnsense
```