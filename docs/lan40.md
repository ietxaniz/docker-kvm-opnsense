# LAN40: Host Services Network

LAN40 represents the **privileged tier** in our network hierarchy, designed to provide trusted virtual machines with comprehensive access to host resources. It addresses the common challenge of integrating development or administrative VMs with services like Docker containers, SSH, and other tools running directly on the host. Instead of using complex workarounds, `LAN40` treats the host as a first-class service provider, accessible via standard networking and secured by OPNsense.

This approach is built on a key architectural feature: **true bidirectional communication**. Unlike other segments where the host can only respond to requests, firewall rules are configured to allow the host to **initiate connections *to*** the VMs on `LAN40`. This creates a fully integrated environment ideal for development, where a VM might run a database that the host needs to access, or for CI/CD tasks where the host needs to deploy applications into a test VM.

The design philosophy is to explicitly grant trust to this dedicated segment rather than weakening overall security. This allows `LAN40` to become a secure administrative zone, perfect for workloads that need to interact with the Docker API, manage the host system, or serve as a trusted point for infrastructure monitoring. This creates a clean and secure separation between your production-like workloads on other LANs and the privileged infrastructure management layer.

## Assigning the Interface

Navigate to `Interfaces - Assignments` and add a new assignment selecting `vtnet5` from the dropdown and adding `LAN40` as description. Next, click on the newly created interface `LAN40`, check **Enable Interface**, select **Static IPv4** as `IPv4 Configuration Type`, set the `IPv4 address` to **192.168.40.1/24**, click `Save`, and then `Apply changes`.

## DHCP Server

Navigate to `Services - Dnsmasq DNS & DHCP - General` and add **LAN40**. Leave the rest of the fields unchanged and click `Apply`.

Then go to `Services - Dnsmasq DNS & DHCP - DHCP ranges`, click the red plus icon, select **LAN40** as interface, set **192.168.40.200** as start address and **192.168.40.240** as end address, set `Subnet mask: 255.255.255.0`, write `LAN40 DHCP` as description, and click `Save` and then `Apply`.

Finally, go to `Services - Dnsmasq DNS & DHCP - DHCP options`, click the red plus icon, and set: `Type: set`, `Option: dns-server [6]`, `Option6: None`, `Interface: LAN40`, `Tag: Nothing selected`, `Value: 8.8.8.8,1.1.1.1`, `Force: unchecked`, `Description: LAN40 DNS = 8.8.8.8,1.1.1.1`, then click `Save` and `Apply`.

## Firewall Rules

Navigate to `Firewall - Rules - LAN40` and add next rules:

- `Action: Pass`, `Interface: Lan40`, `Source: LAN40 net`, `Destination: LAN40 net`, `Description: Allow local access to LAN40 network`.
- `Action: Pass`, `Interface: Lan40`, `Source: LAN40 net`, `Destination: !192.168.0.0/16`, `Description: Allow direct internet access in LAN40`.
- `Action: Pass`, `Interface: Lan40`, `Source: LAN40 net`, `Protocol: any` `Destination: 192.168.99.15`, `Description: Allow full access to host`.

Once three rules are created click on `Apply changes`.

## Testing the Configuration

At this stage, your `LAN40` VMs have privileged **outbound** access to the host and direct access to the internet. We will test this unidirectional connection now. The final step to enable full bidirectional access will be completed in the `LAN99` chapter.

### Test VM-to-Host and Internet Access

From a terminal on your `LAN40` VM, confirm that you can reach the host and the internet:

```bash
ping 192.168.40.1
ping 192.168.99.15
ping 8.8.8.8
curl 192.168.85.15:7080
```

`7080` is the port I defined in my `.env` file for the nginx container on the host. If you've configured a different port, update the command accordingly. For more details, see [readme.md](../services/host/readme.md).

If all is correct, the last command should return:

```html
<html>
    <head>
        <title>host nginx</title>
    </head>
    <body>
        <h1>Access to host web server</h1>
        <p>This demonstrates that you accessed host nginx web server.</p>
    </body>
</html>
```

### Installing services in the VM

Once you can access the host, you can grab the files in `../services/vm-lan40` using sftp or cloning this project in the VM and start them by running `docker compose up`. Run `ip address`  inside the VM to get its IP address, for example `192.168.40.239/24` and check if you can access services in the VM from the host:

`curl 192.168.40.239`

It should return:

```html
<html>
    <head>
        <title>vm nginx</title>
    </head>
    <body>
        <h1>Access to vm web server</h1>
        <p>This demonstrates that you accessed vm nginx web server.</p>
    </body>
</html>
```

