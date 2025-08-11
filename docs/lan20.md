# LAN20: VPN with Port Forwarding using WireGuard

LAN20 demonstrates advanced VPN routing capabilities by combining WireGuard encryption with port forwarding functionality, enabling virtual machines to maintain complete privacy while still accepting incoming connections from the internet. This configuration is ideal for applications that require both anonymity and accessibility, such as peer-to-peer clients, game servers, or self-hosted services that need to be reachable from external networks.

Unlike LAN10's direct internet access, all LAN20 traffic routes through an encrypted WireGuard tunnel to your VPN provider, masking your real IP address and location. The port forwarding capability allows external peers to establish connections directly to services running on LAN20 VMs, with the VPN provider acting as a transparent relay for incoming traffic. This creates a secure, private network segment where services can operate with full internet connectivity while maintaining geographic flexibility and traffic encryption.

In this setup, we'll assign **vtnet3** to a new interface named `LAN20` in OPNsense, configure WireGuard connectivity to a VPN provider that supports port forwarding, and establish the routing policies that direct all LAN20 traffic through the encrypted tunnel. We'll also configure **DHCP services** with VPN-appropriate DNS servers and create **firewall rules** that enable both outbound VPN routing and controlled access to host services through the management network.

The port forwarding configuration requires coordination between the VPN provider's port allocation system and OPNsense's NAT rules, creating a complete path for incoming connections: Internet → VPN Provider → WireGuard Tunnel → OPNsense → VM. This architecture enables applications like BitTorrent clients to maintain full connectivity while routing all traffic through privacy-focused infrastructure.

Once complete, any VM attached to the `virbr-lan20` virtual switch will receive an IP address from OPNsense, route all internet traffic through the WireGuard VPN connection, use VPN provider DNS servers for complete privacy, and support incoming connections through configured port forwards.

## Prerequisites

Before configuring LAN20, ensure you have:

- **VPN provider account** with WireGuard and port forwarding support (tested with OVPN.com)
- **WireGuard configuration file** downloaded from your provider
- **Port forwarding** configured in your provider's dashboard (typically range 49152-65535)

For this guide, we'll use a WireGuard configuration that routes through a VPN provider with port forwarding capabilities. The exact configuration details will vary by provider, but the OPNsense setup process should be consistent across different VPN services.

From the configuration file you should file next table:

| Param    | Value | Description |
| -------- | ----- | ----------- |
| $NAME$   |       | Connection name. I use vpn provider followed by a number |
| $KEY$    |       | private key |
| $PK$     |       | public key  |
| $ADDR$   |       | address     |
| $DNS$    |       | dns servers provided by vpn provider  |
| $ADDRG$  |       | "gateway address, different from $ADDR$ (typically $ADDR$ minus 1, e.g., if $ADDR$ is 172.31.61.210, use 172.31.61.209)" |
| $PPK$    |       | peer public key |
| $PIPS$   |       | peer allowed ips - should be 0.0.0.0/0|
| $PURL$   |       | peer endpoint url |
| $PPORT$  |       | peer endpoint port |
| $FPORT$  |       | forwarded port |

If public key is not included in the configuration file (some vpn providers may not include it) it can be computed with next instruction without leaving any traces of the private key on bash shell:

```bash
(umask 077; echo "<base64-private-key>" | wg pubkey)
history -d $((HISTCMD - 1))
```

If `wg command not found` is answered you need to install wireguard and repeat the previous command:

```bash
sudo apt update && sudo apt install wireguard
```

## Assigning the Interface

Navigate to `Interfaces - Assignments` and add a new assignment selecting `vtnet3` from the dropdown and adding `LAN20` as description. Next, click on the newly created interface `LAN20`, check **Enable Interface**, select **Static IPv4** as `IPv4 Configuration Type`, set the `IPv4 address` to **192.168.20.1/24**, click `Save`, and then `Apply changes`.

## DHCP Server

Navigate to `Services - Dnsmasq DNS & DHCP - General` and in the `Interface` listbox select the networks that will use this service. Add **LAN20**. Leave the rest of the fields unchanged and click `Apply`.

Then go to `Services - Dnsmasq DNS & DHCP - DHCP ranges`, click the red plus icon, select **LAN20** as interface, set **192.168.20.200** as start address and **192.168.20.240** as end address, set `Subnet mask: 255.255.255.0`, write `LAN20 DHCP` as description, and click `Save` and then `Apply`.

### VPN Provider DNS Configuration

For complete privacy and to prevent DNS leaks, LAN20 must use DNS servers provided by your VPN provider rather than your local ISP or public DNS servers. When traffic routes through a VPN tunnel but DNS queries go through your regular internet connection, your browsing activity can still be tracked and associated with your real IP address, defeating the purpose of the VPN.

VPN provider DNS servers ensure that domain name resolution occurs within the encrypted tunnel and exits through the same geographic location as your traffic. This maintains consistency between your apparent location and your DNS queries, preventing correlation attacks and ensuring that geo-restricted content works correctly.

Finally, go to `Services - Dnsmasq DNS & DHCP - DHCP options`, click the red plus icon, and set: `Type: set`, `Option: dns-server [6]`, `Option6: None`, `Interface: LAN20`, `Tag: Nothing selected`, `Value: $DNS$`, `Force: unchecked`, `Description: LAN20 DNS VPN`, then click `Save` and `Apply`.

### Instance configuration

Navigate to `VPN - WireGuard - Instances` and click the **+** button to create a new WireGuard instance. Set next values `Name: $NAME$`, `Public key: $PK$`, `Private key: $KEY$`, `Tunnel address: $ADDR$/32`, `DNS servers: $DNS$`, `Disable Routes: checked`. **DNS servers can be configured when advanced mode is selected**. Leave rest of values untouched. `Save`. (we will enable WireGuard and apply later).

### Peer Configuration

Navigate to `VPN - WireGuard - Peers` and click the **+** button to create the server connection configuration. Set `Name: $NAME$_Peer`, `Public Key: $PPK$`, `Allowed IPs: $PIPS$`, `Endpoint address: $PURL$`, `Endpoint Port: $PPORT$`, `Keepalive Interval: 25` (25 seconds to maintain connection through NAT devices) and finally select `$NAME$` from the `Instances` dropdown list to link this peer to your instance. Click `Save` to create the peer configuration.

### Enable WireGuard Service

Navigate to `VPN - WireGuard`, check **Enable WireGuard** at the bottom of the page, and click `Save`. The WireGuard service will start and attempt to establish the encrypted tunnel to your VPN provider. You can verify the connection status by navigating to `VPN - WireGuard - Status` to see handshake activity and traffic counters.

### Assign WireGuard Interface

Navigate to `Interfaces - Assignments` and assign the WireGuard tunnel as an OPNsense interface. Select the WireGuard interface (e.g., `wg0`) from the dropdown, click the **+** button, and set the description to `$NAME$`. Click on the newly assigned interface, check **Enable Interface**, set **IPv4 Configuration Type** to `None`, set **IPv6 Configuration Type** to `None`, and click `Save` and `Apply Changes`.

### Create Gateway

Navigate to `System - Gateways - Configuration` and click the **+** button to create a static gateway for routing traffic through the VPN tunnel. Set the `Name` to `$NAME$_GW`, select your WireGuard interface from the `Interface` dropdown, set the `IP Address: $ADDRG$`, check **Disable Gateway Monitoring**, and add a description. Click `Save` and `Apply Changes`.

## Configure Outbound NAT

Navigate to `Firewall - NAT - Outbound` and change the mode from "Automatic outbound NAT rule generation" to "Hybrid outbound NAT rule generation" and click `Save`. This allows manual NAT rules to be processed before automatic rules, enabling specific NAT configuration for VPN traffic.

Click the **+** button to create a manual NAT rule for LAN20 traffic. Set `Interface: $NAME$`, `Source address: LAN20 net`, `Destination address: any`, `Translation target: $NAME$ address` and `Description: "NAT LAN20 through VPN tunnel"` and click `Save` and `Apply Changes`.

This NAT rule ensures that traffic from LAN20 is properly translated to use your VPN tunnel IP address, allowing the traffic to flow correctly through the encrypted VPN connection and receive return traffic from remote servers.

## Firewall Rules

Navigate to `Firewall - Rules - LAN20` and add next rules:

- `Action: Pass`, `Interface: Lan20`, `Source: LAN20 net`, `Destination: LAN20 net`, `Description: Allow local access to LAN20 network`.
- `Action: Pass`, `Interface: Lan20`, `Source: LAN20 net`, `Destination: !192.168.0.0/16`, `Gateway: $NAME$_GW`, `Description: Route LAN20 traffic through VPN tunnel`
- `Action: Pass`, `Interface: Lan20`, `Source: LAN20 net`, `Protocol: ICMP` `Destination: 192.168.99.15/32`, `Description: Allow pinging to host`.
- `Action: Pass`, `Interface: Lan20`, `Source: LAN20 net`, `Protocol: TCP/UDP` `Destination: 192.168.99.15/32`, `Destination port range: 445 to 445`, `Description: Allow SAMBA access to host`.

Once four rules are created click on `Apply changes`.

## Fix host access

Execute next instruction.

```bash
sudo nmcli connection modify br-mgmt +ipv4.routes "192.168.20.0/24 192.168.99.1"
sudo nmcli connection reload
sudo nmcli device reapply br-mgmt
```

And remember, if this breaks hosts access to the networks controlled by `opnsense` you will need to switch off `opnsense vm` and then switch it on again (restarting won't work).


## Test the System

To test the configuration, connect the network interface of the vm that you are using for testing to `switch-lan20` and restart it (in orther to load DHCP configurations):

* It receives a valid IP in the `192.168.20.0/24` range
* It can reach the internet
* It can ping the OPNsense LAN20 gateway and the host
* It can access the Samba service at `192.168.99.15`
* Its ip address is different to your routers internet address

In my case, I created a Ubuntu Desktop VM, attached it to the correct virtual network (`switch-lan20`), and powered it on.

Once the system is running, open a terminal and run:

```bash
ip address
ping 192.168.20.1
ping 8.8.8.8
ping 192.168.99.15
curl ifconfig.me
```

## Port Forwarding

This is the last piece we need to implement for completing this chapter. This requires requesting port forwarding from our vpn provider (opvn.com in my case), and setting the corresponding firewall rules one in `NAT` and the other in `LAN20`. I will use `$FPORT$`.

Navigate to `Firewall - NAT - Port Forward`, and add a new rule with next values: `Interface: $NAME$`, `Protocol: TCP/UDP`, `Source: any`, `Destination: $NAME$ address`, `Destination port range: $FPORT$ to $FPORT$`, `Redirect target ip: 192.168.20.5`, `Redirect target port: $FPORT$`, `Description: 192.168.20.5:$FPORT$ forwarded`. `Save` and `Apply changes`.

Navigate to `Firewall - Rules - $NAME$`, delete automatic created rule and add a new rule with next values: `Interface: $NAME$`, `Protocol: TCP/UDP`, `Source: any`, `Destination: 192.168.20.5`, `Destination port range: $FPORT$ to $FPORT$`, `Reply to: $NAME$_GW` (**in advanced features**), `Description: 192.168.20.5:$FPORT$ forwarded`.

## Test Port Forwarding

To ensure port forwarding always targets the correct machine, we assign a **static IP** to the VM running the service. This is done by reserving the IP address based on the VM’s MAC address in the DHCP server configuration.

Navigate to `Services - Dnsmasq DNS & DHCP - Leases`, find the VM you want to use for port forwarding, and click the small icon on the right side of the row. This will either take you directly to an `Edit Host Override` window or redirect you to the Hosts list, in which case you need to click the pencil icon next to the newly created entry to open the override editor.

Inside the `Edit Host Override` window, set the IP address to a static value — in this case, `192.168.20.5`. Leave the MAC address and hostname as-is. Click `Save`, and then click `Apply` at the top of the page.

Restart the VM and verify the new IP assignment by running `ip address` in the VM terminal. It should now have the static IP `192.168.20.5`. To determine the external VPN IP address (which will be used for testing), run `curl ifconfig.me` from the VM. This will return the public IP address assigned to your WireGuard tunnel.

Now start a listener on the VM that waits for incoming connections on the forwarded port. Execute:

```bash
nc -lvkp $FPORT$
```

Leave this terminal open and listening.

Then, from another machine located outside your local network — for example, from the host itself using its direct internet connection run:

```bash
nc -vz <vpn-ip-address> $FPORT$
```

If the connection is successful, it will respond with:

```
Connection to <vpn-ip-address> $FPORT$ port [tcp/*] succeeded!
```

This confirms that your port forwarding is working correctly end-to-end: from the external network through your VPN provider, into the WireGuard tunnel, through OPNsense, and finally to the intended VM.
