# Architecture Overview

The homelab architecture centers around network segmentation using Debian 13 KDE as the host system with KVM virtualization and OPNsense managing routing and firewall functions. The design isolates different network segments while maintaining centralized control through a single OPNsense virtual machine that connects to each network segment through dedicated interfaces.

The system uses **three distinct networking layers** that work together to provide full functionality and isolation:

1. **External Connectivity Layer**: This layer provides WAN access for both the host and OPNsense. It’s implemented through a bridge named `br0`, which connects to the physical interface `enp1s0` and inherits its MAC address. This design ensures the home router continues to recognize the system under the same MAC, preserving DHCP leases and port forwarding rules for both the host and the OPNsense VM.

2. **Management Layer**: This layer provides administrative access to the OPNsense firewall and controlled access to host-based services like Docker containers. It’s implemented via the `br-mgmt` bridge on the `192.168.99.0/24` subnet. The host itself participates in this network with the IP `192.168.99.15`, while OPNsense serves as the gateway at `192.168.99.1`.

3. **Segmented VM Layer**: This consists of isolated virtual switches (`virbr-lan10`, `virbr-lan20`, etc.) created and managed by libvirt. Each switch represents a distinct virtual LAN, with OPNsense acting as the default gateway and enforcing routing, firewall, and VPN policies for each segment. All inter-segment traffic is routed through OPNsense, preserving strict Layer 2 separation between VM networks.

![](../figs/architecture.drawio.svg)

## VM Network Segments and Routing Policies

Each VM network segment operates as an isolated virtual switch managed by libvirt, with connectivity and internet access controlled entirely by OPNsense routing policies. The LAN10 segment provides direct internet access through the WAN interface, suitable for applications requiring maximum performance without VPN overhead. LAN20 implements WireGuard VPN routing with port forwarding capabilities, designed for applications that need both privacy and incoming connections.

LAN30 demonstrates dynamic VPN capabilities by supporting multiple WireGuard endpoints that can be switched programmatically, allowing the same network segment to route through different geographic locations or VPN providers based on current requirements. LAN40 serves as a privileged network segment for trusted VMs that need full bidirectional access to host services such as Docker, Samba, or SSH. LAN50 operates as a completely isolated network with no internet access, ideal for secure development and testing workloads.

## OPNsense Central Router

The OPNsense virtual machine connects to all three network layers through dedicated virtual interfaces. The WAN interface is connected via a virtio NIC that bridges directly to br0 using libvirt. This design ensures that OPNsense receives an IP address from the home router via DHCP and can route traffic externally, without requiring any internal bridging inside OPNsense itself. All bridging and MAC inheritance are handled at the host level using NetworkManager, keeping the OPNsense configuration minimal and predictable. The management interface connects to br-mgmt for administrative functions, which is firewall-restricted by OPNsense to prevent lateral movement into restricted segments. Five additional interfaces connect to the isolated virtual switches virbr-lan10, virbr-lan20, virbr-lan30, virbr-lan40, and virbr-lan50, with each interface serving as the gateway for its respective network segment.

OPNsense manages DHCP services for each network segment, provides firewall rules that control traffic between segments, and implements the VPN connections that route traffic through different providers or geographic locations. The centralized approach means that all routing decisions, security policies, and network monitoring occur within OPNsense, while the underlying virtual switches simply provide isolated communication channels between virtual machines and the central router.

## Host Integration and Service Access

The host system participates in the management network through the br-mgmt bridge, receiving the IP address 192.168.99.15 and using OPNsense at 192.168.99.1 as the gateway for reaching other network segments. This configuration enables virtual machines on isolated network segments to access Docker containers and other services running on the host through firewall-controlled connections managed by OPNsense.

Docker containers operate on the 172.16.0.0/12 address space and remain accessible to authorized network segments through static routes configured on the host using the `ports` field in a docker compose file, for example. These routes direct traffic destined for the various network segments through OPNsense, while OPNsense firewall rules control which segments can access which host services, maintaining security boundaries while enabling practical service integration.

## Virtual Machine Placement and Mobility

Virtual machines connect to network segments by attaching to the corresponding libvirt virtual switch, similar to plugging an Ethernet cable into a physical switch. A gaming VM with GPU passthrough might connect to `virbr-lan10` for direct internet access to minimize latency, while a p2p VM connects to `virbr-lan20` to route all traffic through a VPN with port forwarding capabilities.

Virtual machines can be moved between network segments by changing their virtual switch attachment, instantly changing their routing behavior without requiring reconfiguration of the VM itself. This flexibility enables testing different routing policies, switching between VPN providers, or isolating workloads for security purposes by simply reassigning network connections through the KVM management interface. OPNsense automatically provides appropriate IP addressing and routing policies based on which virtual switch each VM connects to.

Beyond individual workloads, this architecture also excels as a powerful testing environment for distributed systems. For instance, one could simulate a multi-region deployment by placing 'Node A' of a distributed database on `LAN10` (representing a local datacenter), 'Node B' on `LAN20` (routed via a US VPN), and 'Node C' on `LAN30`. It's an ideal environment for studying consensus algorithms like Raft, testing the resilience of a distributed key-value store like etcd, testing service discovery, or verifying database replication setups such as PostgreSQL logical replication or a clustered rqlite database, especially when using the switchable VPN on LAN30 to simulate dynamic IP changes. I also use this environment to test mTLS gRPC services for a cloud-to-edge IoT platform I'm building, including secure gateway communication and backend coordination.

Virtual machines in this architecture can switch between network segments dynamically using two supported approaches. One option is to attach multiple virtual NICs to a VM, each connected to a different virtual switch (e.g., LAN10 and LAN20), allowing the guest OS or OPNsense to control which interface is active via routing policies or interface metrics. Alternatively, a VM can use a single NIC and be moved between segments by hot-unplugging it from one virtual network and hot-plugging it into another using `virsh` or `virt-manager`. This does not require rebooting the VM, and the new interface is detected dynamically by the guest. Both methods are valid: multi-homing provides flexibility for advanced setups, while hot-switching offers simplicity and clarity for single-path routing scenarios.

## Hardware Considerations & Real-World Experience

I developed and tested this procedure on a Ryzen 7 8700G system with 128 GB of RAM and an RTX 4060 Ti used exclusively for passthrough. For a build like this, it’s important to note that **abundant RAM is often more critical than raw CPU power**. The 128 GB is generous, but a large allocation is essential for running many VMs simultaneously, as each requires its own dedicated memory. In contrast, CPU performance only becomes a bottleneck for compute-heavy tasks like gaming or compiling inside VMs.

While the 8700G works as intended, newer CPUs may still suffer from early-stage driver quirks — particularly around iGPU behavior. In this setup, stability issues were ultimately traced back to the BIOS VRAM setting: reserving 16 GB for the iGPU caused persistent instability, especially under KDE. Reducing it to 4 GB resolved the problem entirely.

If you're building a similar system, older APUs like the 7700G or 5700G may offer better stability out of the box. Intel is also a solid option. Since the iGPU in this design only handles the host display (while all GPU-intensive workloads run in passthrough VMs), Intel’s mature drivers and lower idle power usage make them appealing.

And for those wondering — **KDE works beautifully once stable**. It’s a powerful desktop environment and far more productive than GNOME.
