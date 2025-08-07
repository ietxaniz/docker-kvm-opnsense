# VM Startup Orchestrator

In this section we’ll create a simple startup orchestrator that ensures our virtual machines only boot once the network infrastructure — specifically the OPNsense firewall — is fully online. This avoids race conditions where VMs boot too early, miss DHCP, or fail service discovery because routing isn't up yet.

The orchestrator will be implemented as a `systemd` service that starts automatically on boot. It will first launch the OPNsense VM using `virsh`, then wait for the OPNsense web interface (`http://192.168.99.1`) to become available. Only after that will it proceed to launch the rest of the VMs defined in a configuration file. This setup ensures clean, deterministic startup for all dependent workloads.

We’ll separate the configuration into two files. One contains the name of the OPNsense VM, and the other lists the remaining VMs to launch after the network is ready. This allows the orchestrator logic to remain generic and gives you flexibility to manage machine sets independently.

## Configuration Files

We’ll place the configuration under `/etc/vm-orchestrator`.

First, create the directory:

```bash
sudo mkdir -p /etc/vm-orchestrator
```

Then define the OPNsense VM in a file called `opnsense.conf`. This should contain a single environment-style variable:

```bash
echo 'OPNSENSE_VM=opnsense' | sudo tee /etc/vm-orchestrator/opnsense.conf
```

Now create the list of dependent VMs. Each line should contain the name of a VM managed by `virsh`. You can use any names you defined previously when creating the machines:

```bash
sudo tee /etc/vm-orchestrator/vms.conf <<EOF
dev-ubuntu
vpn-client
ci-node1
EOF
```

## Orchestrator Script

Next we’ll write the actual orchestrator logic. This script is placed in `/usr/local/bin` and will be invoked by the systemd unit.

Create the file:

```bash
sudo nano /usr/local/bin/vm-orchestrator.sh
```

Paste the following content:

```bash
#!/bin/bash
set -e

CONFIG_DIR="/etc/vm-orchestrator"
OPNSENSE_CONF="$CONFIG_DIR/opnsense.conf"
VM_LIST="$CONFIG_DIR/vms.conf"

if [[ ! -f "$OPNSENSE_CONF" ]]; then
    echo "Missing $OPNSENSE_CONF"
    exit 1
fi
source "$OPNSENSE_CONF"

if [[ -z "$OPNSENSE_VM" ]]; then
    echo "OPNSENSE_VM not defined in $OPNSENSE_CONF"
    exit 1
fi

echo "[orchestrator] Starting OPNsense VM: $OPNSENSE_VM"
virsh start "$OPNSENSE_VM" || true

echo "[orchestrator] Waiting for http://192.168.99.1 to become available..."
until curl -sf --max-time 1 http://192.168.99.1 > /dev/null; do
    sleep 2
done
echo "[orchestrator] OPNsense is up."

if [[ ! -f "$VM_LIST" ]]; then
    echo "[orchestrator] No VM list found at $VM_LIST"
    exit 0
fi

while IFS= read -r vm_name; do
    [[ -z "$vm_name" ]] && continue
    echo "[orchestrator] Starting VM: $vm_name"
    virsh start "$vm_name" || true
done < "$VM_LIST"
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/vm-orchestrator.sh
```

## Systemd Service

Now we’ll define the systemd unit that runs the orchestrator at boot.

Create the service file:

```bash
sudo nano /etc/systemd/system/vm-orchestrator.service
```

Paste the following:

```ini
[Unit]
Description=VM Startup Orchestrator
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/vm-orchestrator.sh
RemainAfterExit=true

[Install]
WantedBy=multi-user.target
```

Enable the service so it runs at boot:

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable vm-orchestrator.service
```

You can test it without rebooting by running:

```bash
sudo systemctl start vm-orchestrator.service
```

If everything is working, you should see your OPNsense VM start, followed by a pause, and then each listed VM starting in order once `http://192.168.99.1` becomes reachable.

This completes the orchestrator setup. Your system will now boot cleanly, wait for network readiness, and launch VMs in a predictable and reliable sequence. You can extend this later with per-VM timeouts, grouped startup tiers, or hooks to execute custom actions after boot. But in its current form, it already solves the core issue: ensuring that network-dependent VMs never boot prematurely.
