# Host Access to Isolated LANs (and Back)

In this architecture, the host system is connected only to the **management network** (`br-mgmt`) with the IP address `192.168.99.15`. It is not directly attached to any of the OPNsense-managed LANs — such as `LAN10`, `LAN20`, etc. — by design. This provides clean separation between infrastructure and workloads, with all inter-network traffic routed through OPNsense.

However, this setup introduces a routing limitation: when a virtual machine on an isolated LAN attempts to access a service running on the host (such as Samba), the request reaches the host successfully via OPNsense — but **the host doesn’t know how to reply**, because it has no route back to that VM’s network. The result is a silent failure: the VM connects out, but the host’s response is misrouted and lost.

This document explains why this happens and how to resolve it by adding persistent static routes to the host system — restoring full **bidirectional communication between the host and any segmented LAN** routed through OPNsense.

## Why Host Replies Fail

OPNsense acts as a multi-interface router and firewall, handling routing between its interfaces (`LAN10`, `LAN20`, etc.) and the outside world (`WAN`). When a VM on `LAN10` tries to connect to the host at `192.168.99.15`, the path is:

```
VM → LAN10 (vtnet2) → OPNsense → LAN99 (vtnet5) → Host
```

This forward path works because OPNsense knows how to route between LAN interfaces. But when the host receives the connection, it attempts to respond. The problem is: the host has **no route back** to `192.168.10.0/24` (or any other LAN), so it uses the system default — which points to your physical router via `br0`. That response is lost.

## The Solution: Static Routes on the Host

The host must be explicitly told:

> *"To reach `192.168.10.0/24`, send traffic through `192.168.99.1` (the OPNsense interface on br-mgmt)."*

This is done by adding **per-network static routes** to the `br-mgmt` interface, using NetworkManager. These routes are persistent and will survive reboots:

```bash
sudo nmcli connection modify br-mgmt +ipv4.routes "192.168.10.0/24 192.168.99.1"
sudo nmcli connection modify br-mgmt +ipv4.routes "192.168.20.0/24 192.168.99.1"
sudo nmcli connection modify br-mgmt +ipv4.routes "192.168.30.0/24 192.168.99.1"
sudo nmcli connection modify br-mgmt +ipv4.routes "192.168.40.0/24 192.168.99.1"
sudo nmcli connection modify br-mgmt +ipv4.routes "192.168.50.0/24 192.168.99.1"
sudo nmcli connection up br-mgmt
```

Once these are applied, the host has complete routing awareness for all OPNsense-managed LANs. Any return traffic (such as SMB replies, API responses, or ping replies) will now go back through the correct path.

## Why This Happens: A Routing Ambiguity

By default, the host’s routing table includes only:

* Its directly attached networks (like `192.168.99.0/24`)
* A default route (`0.0.0.0/0`) pointing to your physical router

So when the host receives a packet from `192.168.10.12`, it has no idea where that network lives. Without an explicit route, it assumes the default path — which is incorrect in this case.

Adding a static route for each LAN resolves this ambiguity. The host now sees that traffic for `192.168.10.0/24` should go through `192.168.99.1`, which ensures return packets reach OPNsense and then the original VM.

## Why We Need Multiple Routes (Not One)

All your segmented LANs currently reside in the `192.168.0.0/16` range. Since your host also uses part of that range for its own network (e.g. `192.168.99.0/24`), you **cannot safely summarize** the LANs into a single route like `192.168.0.0/16` — doing so would also match your actual physical LAN or internet access and cause routing conflicts.

The only safe solution in this case is to define **explicit static routes**, one per LAN:

```bash
sudo nmcli connection modify br-mgmt +ipv4.routes "192.168.10.0/24 192.168.99.1"
```

Repeat this for each additional LAN (`192.168.20.0/24`, `192.168.30.0/24`, etc.).

This is necessary because of **overlapping address space** between virtual and physical networks — the host cannot distinguish which `192.168.x.x` subnets are internal or external without precise instructions.

---

## Alternate Design 1: Summary Routing with `10.10.0.0/16`

To simplify the setup, you can migrate **all OPNsense-managed LANs and the host's management network** into the `10.10.0.0/16` block.

* Use `10.10.10.0/24`, `10.10.20.0/24`, etc. for LAN segments
* Assign **OPNsense's management IP** as `10.10.99.1/24`
* Assign the **host's `br-mgmt` IP** as `10.10.99.15/24`
* Then add a single summary route to reach all LANs:

```bash
sudo nmcli connection modify br-mgmt ipv4.routes "10.10.0.0/16 10.10.99.1"
```

This design avoids overlapping with common home LANs and allows you to reach all isolated subnets with a single, clean route.

---

## Alternate Design 2: Everything Inside `10.10.0.0/16` (No Static Routes at All)

This is the cleanest possible design: place **everything**, including the host and all virtual LANs, inside the same `10.10.0.0/16` network.

* Host’s `br-mgmt` IP: `10.10.99.15/16`
* OPNsense management interface: `10.10.99.1/24`
* LAN10: `10.10.10.0/24`, LAN20: `10.10.20.0/24`, etc.

Now the host sees the entire `10.10.0.0/16` block as **directly connected** and doesn’t need any static routes at all. The routing “just works” — replies from the host to any VM go back through the same interface they came from.

OPNsense continues to isolate traffic between LANs using its own firewall rules, so this design keeps all segmentation and control intact.

---

## Summary

| Design                                | Needs static routes? | Summary routing? | Risk of overlap? |
| ------------------------------------- | -------------------- | ---------------- | ---------------- |
| `192.168.X.X` LANs + host             | ✅ Yes (one per LAN)  | ❌ No             | ⚠️ Yes           |
| All in `10.10.0.0/16` w/ static route | ✅ Yes (single route) | ✅ Yes            | ✅ No             |
| **All in `10.10.0.0/16` (full move)** | ❌ No                 | ✅ Inherent       | ✅ No             |

But in our current network design, to allow VMs on isolated LANs to access host services — and for the host to respond — you must:

* Route all host replies through the OPNsense gateway on `br-mgmt`
* Use `nmcli` to define persistent static routes for each virtual LAN
* Preserve OPNsense as the sole bridge between LANs and host

Think of the host as being connected to **two internets**:

1. The **public internet**, reachable via `br0` and your physical router
2. A **private internet**, managed by OPNsense, containing all segmented LANs

By default, the host routes everything it doesn't recognize to the public internet. Adding these static routes simply tells the host, *“This traffic actually belongs to your private internet — send it to OPNsense.”*

Once done, host ↔ VM communication works as intended — without breaking network isolation or introducing insecure shortcuts.
