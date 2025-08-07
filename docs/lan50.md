# LAN50: Isolated Development Network

LAN50 is an **offline-only** network segment intended for development, testing, and secure workloads that should not have internet access. Virtual machines inside this network are fully isolated from the outside world — but have **complete access to each other** for collaboration, internal service testing, or multi-node environments.

Unlike other segments, LAN50 enforces **partial host access**: only the **first 64 IP addresses** in the subnet (`192.168.50.1` through `192.168.50.63`) are allowed to **ping or access Samba services on the host**. All other addresses in LAN50 can still communicate freely with each other but will be blocked from reaching host-based services. This setup allows you to tightly control which internal machines are allowed to interact with the host, without compromising intra-network flexibility.

LAN50 is ideal when you need a safe space to run experiments, spin up VM clusters, or perform testing where internet access is not required or should be explicitly blocked.


## Assigning the Interface

Navigate to `Interfaces - Assignments` and add a new assignment selecting `vtnet6` from the dropdown and adding `LAN50` as description. Next, click on the newly created interface `LAN50`, check **Enable Interface**, select **Static IPv4** as `IPv4 Configuration Type`, set the `IPv4 address` to **192.168.50.1/24**, click `Save`, and then `Apply changes`.


## DHCP Server

Navigate to `Services - Dnsmasq DNS & DHCP - General` and add **LAN50**. Leave the rest of the fields unchanged and click `Apply`.

Then go to `Services - Dnsmasq DNS & DHCP - DHCP ranges`, click the red plus icon, select **LAN50** as interface, set **192.168.50.200** as start address and **192.168.50.240** as end address, set `Subnet mask: 255.255.255.0`, write `LAN50 DHCP` as description, and click `Save`, then `Apply`.


## Firewall Rules

Navigate to `Firewall - Rules - LAN50` and add the following rules:

* `Action: Pass`, `Interface: LAN50`, `Source: LAN50 net`, `Destination: LAN50 net`,
  `Description: Allow unrestricted LAN50 internal traffic`.

* `Action: Pass`, `Interface: LAN50`, `Source: 192.168.50.0/26`, `Protocol: ICMP`, `Destination: 192.168.99.15/32`,
  `Description: Allow ping to host from 192.168.50.1 to 192.168.50.63`.

* `Action: Pass`, `Interface: LAN50`, `Source: 192.168.50.1-192.168.50.50`, `Protocol: TCP/UDP`,
  `Destination: 192.168.99.15/32`, `Destination port range: 445 to 445`,
  `Description: Allow Samba access to host from lower LAN50 IPs`.

* `Action: Block`, `Interface: LAN50`, `Source: LAN50 net`, `Destination: !LAN50 net`,
  `Description: Block outbound traffic to non-LAN50 networks`.

Click `Apply changes` after creating the rules.


## Testing the Configuration

Launch two VMs connected to `switch-lan50`:

* One with an IP like `192.168.50.25`
* Another with an IP like `192.168.50.201`

From both VMs, run:

```bash
ping 192.168.50.1
ping 192.168.50.200
ping 192.168.99.15
```

### Expected Results

| Test                            | `192.168.50.25` | `192.168.50.201` |
| ------------------------------- | --------------- | ---------------- |
| Ping gateway (`192.168.50.1`)   | Yes             | Yes              |
| Ping another VM                 | Yes             | Yes              |
| Ping host (`192.168.99.15`)     | Yes             | No               |
| Access Samba on host (port 445) | Yes             | No               |

This confirms full internal reachability within LAN50, and that selective host access rules are enforced correctly based on source IP.
