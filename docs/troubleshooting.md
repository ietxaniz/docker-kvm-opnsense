# Troubleshooting

## Troubleshooting: OPNsense Rule Editor Gotchas

OPNsense’s rule editor has several unintuitive UI elements that can easily lead to misconfiguration. Below are common areas where I often get stuck when creating or editing firewall rules.

### Selecting Network Objects

When setting **Source** or **Destination** to predefined networks like `LAN99 net` or `LAN99 address`, OPNsense presents a multi-select list in a small dropdown window.

Always uncheck `any` first before selecting your intended network(s). Then check `LAN99 net` (or whichever network you're targeting). Due to the limited height, scroll carefully and confirm your selection before saving.

### Defining a Single Host IP

To enter a specific IP address (like `192.168.99.15`), first select `Single host or Network` from the dropdown in the **Source** or **Destination** field. Only after that selection will an input box appear where you can type the IP.

This behavior is not immediately obvious, so be sure to watch for the input field appearing below the dropdown.

### Specifying Custom Ports

To set a custom port or port range (for example, `445` for Samba), you need to:

1. Select `other` from the port dropdown.
2. Enter the port number manually in the text field below.

To match a single port, input the same value in both `from` and `to` fields.

### Service in other subnet responds to ping but not to other protocols (e.g. Samba)

If a service (like the host machine) is reachable via ping but hangs on TCP connections (e.g. Samba, SSH, HTTP), check the subnet mask assigned by DHCP in `Services - Dnsmasq DNS & DHCP - DHCP ranges`. If its marked as **automatic** set it to `255.255.255.0` or to the corresponding setting if you are using a different architecture. For instance if a VM receives a `255.255.0.0` mask instead of the expected `255.255.255.0`, it may assume other subnets (like `192.168.99.0/24`) are directly reachable — bypassing OPNsense entirely. This breaks connection state tracking and causes replies to be silently dropped. Always set the DHCP subnet mask explicitly.

### NTP appears as “not running” in Lobby Dashboard

If the Lobby Dashboard shows NTP as “not running”, and restarting the service doesn't help, the issue may be caused by WireGuard interfaces defined in the system.

By default, OPNsense tries to bind NTP to all interfaces — including any WireGuard tunnels. This can lead to socket conflicts and cause the service to crash with `signal 11` errors.

To fix it, restrict NTP to only use the WAN interface:

Navigate to:

```
Services → Network Time → General
```

In the **Interface** listbox, select only **WAN**. Click **Save**.

This prevents NTP from binding to WireGuard interfaces and allows the service to start normally.

