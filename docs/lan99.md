# LAN99: Management and Host Services Network

LAN99 is the dedicated **management segment** that connects the host system to the virtual network architecture. It is the only network where the host has a direct presence, using the IP `192.168.99.15` via the `br-mgmt` bridge. This segment is used for administrative access to OPNsense (web and SSH), and to expose host-based services such as Docker containers or Samba shares to trusted virtual machines.

By default, OPNsense assigns the first LAN interface as `LAN`, operating on the `192.168.99.0/24` network and using `vtnet1`. We renamed this interface to `LAN99` and configured it with the static address `192.168.99.1`. The host is configured manually with `192.168.99.15`. No DHCP server is needed on this network.

Initially, OPNsense creates a permissive firewall rule allowing all traffic from LAN99. We will now **lock down this segment** to limit its privileges. The goal is to allow LAN99 to:

* Access OPNsense for admin purposes (web + SSH)
* Reach host services such as Samba
* Communicate with LAN40 VMs for trusted development workflows

All other traffic will be explicitly blocked. This turns LAN99 into a **privileged but scoped zone**, maintaining administrative access without exposing unnecessary paths.


## Firewall Rules

Navigate to `Firewall - Rules - LAN99` and locate the default IPv4 rule that allows all traffic.

Edit this rule by changing **Destination** from `any` to `LAN99 net`. In the dropdown, uncheck `any`, scroll, and select `LAN99 net`. Leave all other fields unchanged, then click `Save`.

Next, delete the default IPv6 rule (if present), as IPv6 is disabled in this setup.

Now add the following rules, in order:

* `Action: Pass`, `Protocol: ICMP`, `Source: any`, `Destination: 192.168.99.15/32`,
  `Description: Allow ping to host`

* `Action: Pass`, `Protocol: TCP`, `Source: any`, `Destination: 192.168.99.15/32`, `Destination port range: 445 to 445`,
  `Description: Allow Samba access to host`

* `Action: Pass`, `Interface: LAN99`, `Source: LAN99 net`, `Destination: LAN40 net`,
  `Description: Allow access to LAN40`

Click **Apply changes** to activate the updated rules.

## How This Relates to LAN40

In the `LAN40` chapter, we established that VMs in LAN40 can access the host — and the host can access those VMs — thanks to permissive firewall rules on the `LAN40` interface. However, **host traffic back to LAN40 VMs also traverses the LAN99 interface**, because that’s the only interface the host is directly connected to.

That means **LAN99 must explicitly allow outbound access to LAN40**, or the host will be blocked from reaching trusted VMs.

This setup creates a **layered security model**:

* `LAN40` defines what the VMs are allowed to do
* `LAN99` defines what the host is allowed to access — acting as the final gatekeeper

This ensures that even if another network (e.g., LAN10) misconfigures its rules and somehow reaches the host, it will still be blocked at `LAN99`.

## Test the Configuration

From the host, verify the expected connectivity:

```bash
ping 192.168.99.1
ping 192.168.40.1
```

Expected behavior:

* `ping 192.168.99.1` confirms OPNsense is reachable from the host
* `ping 192.168.40.1` confirms that the host can reach LAN40 — meaning both rules (LAN40 and LAN99) are functioning

## Final Notes

LAN99 now acts as the **administrative backbone** of your homelab — directly connected to the host, scoped to interact with trusted resources like LAN40, and explicitly secured through firewall rules that block all other paths. This completes the trust chain between the host and the segmented networks, ensuring that **only authorized traffic flows into or out of infrastructure-critical components**.
