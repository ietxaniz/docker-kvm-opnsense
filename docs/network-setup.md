# Network Setup

## Docker host access

After updating and upgrading my fresh installation, the first thing I do is to install Docker. This is a foundational step for our homelab architecture. The main purpose is to enable communication between our future virtual machines (managed by OPNsense) and services running in Docker containers on the host itself. For instance, you could have a Samba container for file sharing or a web server container on the host, and this network setup will allow us to create firewall rules in OPNsense so your VMs can access them. This integrates the host's container ecosystem with the virtualized network. In the future I will be adding more containers and networks to docker, and I usually use `172.16.0.0/12` addresses exclusively for docker.

So if we execute `ip address` in console, we will see that we have the original networks and the docker default network called `docker0`. In my machine:

```bash
ip address
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:18:44:d2 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.175/24 brd 192.168.1.255 scope global dynamic noprefixroute enp1s0
       valid_lft 85129sec preferred_lft 85129sec
    inet6 fe80::5054:ff:fe18:44d2/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 46:e0:0b:c1:88:85 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

## Create br0 Network

`br0` will be used as main WAN interface for opnsense VM. This will allow opnsense to have direct internet connection without interfering with the host.

Before creating the network we will get some information such as mac address, the name of the connection in NetworkManager and ip address of the device. For example in my machine I get next results:

```bash
inigo@deb12-gnome-testing:~$ ip link show enp1s0 | grep "link/ether"
    link/ether 52:54:00:18:44:d2 brd ff:ff:ff:ff:ff:ff
inigo@deb12-gnome-testing:~$ nmcli connection show --active
NAME                UUID                                  TYPE      DEVICE  
Wired connection 1  2c13c152-7d32-4578-88c0-68fe51afa496  ethernet  enp1s0  
lo                  73b9ef06-eaae-4a30-8007-7e3d049e4328  loopback  lo      
docker0             80ff95ee-a778-4ffb-bc0a-88d920053afe  bridge    docker0 
inigo@deb12-gnome-testing:~$ ip addr show enp1s0
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:18:44:d2 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.175/24 brd 192.168.1.255 scope global dynamic noprefixroute enp1s0
       valid_lft 86158sec preferred_lft 86158sec
    inet6 fe80::5054:ff:fe18:44d2/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```

So I need to store the mac address, and I will annotate it in the document `network-data.md`. (you should have your own table to continue with this tutorial)

| Parameter                   | Value                              |
| --------------------------- | ---------------------------------- |
| host-connection-name        | Wired connection 1                 |
| host-mac                    | 52:54:00:18:44:d2                  |


Before running verification commands we will need to install `bridge-utils`, so execute in shell next command: `sudo apt update && sudo apt install bridge-utils` now, as if we lose internet we will not be able to do it later.


### Emergency rollback

Before creating the bridge you should have at hand next instructions to recover internet connection in case you lose connectivity.

```bash
nmcli connection down br0 2>/dev/null || true
nmcli connection delete br0 2>/dev/null || true
nmcli connection delete bridge-slave-enp1s0 2>/dev/null || true

nmcli connection add type ethernet con-name "Wired connection 1" ifname enp1s0 \
  ipv4.method auto ipv6.method ignore connection.autoconnect yes

nmcli connection up "Wired connection 1"

ping -c 2 8.8.8.8
```

### Create the br0 Brige with MAC Inheritance

```bash
nmcli connection down "Wired connection 1"
nmcli connection delete "Wired connection 1"

nmcli connection add type bridge con-name br0 ifname br0 \
  ipv4.method auto \
  ipv6.method ignore \
  802-3-ethernet.cloned-mac-address 52:54:00:18:44:d2 \
  bridge.stp no \
  connection.autoconnect yes

nmcli connection add type ethernet con-name bridge-slave-enp1s0 ifname enp1s0 \
  connection.master br0 \
  connection.slave-type bridge \
  connection.autoconnect yes

nmcli connection up bridge-slave-enp1s0

nmcli connection up br0

sleep 5
```

### Verify Bridge Creation

Run these diagnostic commands to confirm the bridge is working properly:

```bash
echo "=== BRIDGE DIAGNOSTICS ==="
ping -c 2 8.8.8.8
ip addr show br0
/usr/sbin/brctl show br0
echo "br0 carrier: $(cat /sys/class/net/br0/carrier 2>/dev/null || echo 'unknown')"
echo "enp1s0 carrier: $(cat /sys/class/net/enp1s0/carrier 2>/dev/null || echo 'unknown')"
nmcli device status | grep -E "(br0|enp1s0)"
echo "=== END DIAGNOSTICS ==="
```

After successful bridge creation, you should see:

- Succesful ping response to 8.8.8.8
- `br0` interface with an IP address in the 192.168.1.0/24 range (same as your original enp1s0)
- `brctl show br0` displaying enp1s0 as a member of the bridge
- Both `br0` and `enp1s0` showing as "connected" in nmcli device status.

## Create br-mgmt Bridge

Let's create the management bridge that will be used for the OPNsense management network:

```bash
nmcli connection add type bridge con-name br-mgmt ifname br-mgmt \
  ipv4.method manual \
  ipv4.addresses 192.168.99.15/24 \
  ipv4.never-default yes \
  ipv6.method ignore \
  bridge.stp no \
  connection.autoconnect yes

nmcli connection up br-mgmt
```

This will create the network and start it up. This commands are persistent so this will work till you command to stop or delete it.

## Verify br-mgmt Creation

Run these diagnostic commands to confirm the br-mgmt bridge is working properly:

```bash
echo "=== BR-MGMT DIAGNOSTICS ==="
ping -c 2 192.168.99.15
ip addr show br-mgmt
/usr/sbin/brctl show br-mgmt
echo "br-mgmt carrier: $(cat /sys/class/net/br-mgmt/carrier 2>/dev/null || echo 'unknown')"
nmcli device status | grep br-mgmt
nmcli connection show | grep br-mgmt
echo "Default route (should still be through br0):"
ip route show | grep default
echo "=== END DIAGNOSTICS ==="
```

After successful br-mgmt creation, you should see:

- **Successful ping response** to 192.168.99.15 (the bridge can ping itself)
- **br-mgmt interface** with IP address 192.168.99.15/24 and state UP
- **brctl show br-mgmt** displaying an empty bridge (no interfaces attached yet)
- **br-mgmt carrier: unknown** (normal for empty bridge with no connected devices)
- **nmcli device status** showing br-mgmt as "connected"
- **nmcli connection show** showing br-mgmt bridge connection
- **Default route** still pointing through br0 (not interfering with internet)

The br-mgmt bridge should be ready to accept other kvm VM-s to connect to it. This bridge will have two purposes in the future: serve as interface for connecting to opnsense management (both via web and ssh) and serve as the network for sharing services running in the host to the machines connected to opnsense network.

