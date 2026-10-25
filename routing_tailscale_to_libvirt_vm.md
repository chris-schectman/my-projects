# Routing Tailscale Traffic to a Libvirt/KVM Virtual Machine on Fedora
October, 2026

---

If you run virtual machines on a Fedora host using Virtual Machine Manager (libvirt/KVM) and want to access them remotely via Tailscale, you might run into a wall. Even after advertising the subnet and tweaking `firewalld`, your pings might fail with a "Destination Port Unreachable" error. 

Here is the complete, step-by-step guide to successfully routing Tailscale traffic into a libvirt NAT network on a modern Fedora host, including how to bypass libvirt's strict, hidden firewall rules.

## Step 1: Enable IP Forwarding
Tailscale requires IP forwarding to be enabled in the Linux kernel to act as a subnet router. Fedora disables this by default.

Enable IPv4 and IPv6 forwarding permanently:

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

## Step 2: Advertise the VM Subnet
Tell Tailscale to broadcast your virtual machine network (for example, `192.168.100.0/24`).

```bash
sudo tailscale up --advertise-routes=192.168.100.0/24 --accept-routes
```

**Crucial Step:** Tailscale blocks new subnet routes by default for security. 
1. Log into the Tailscale Admin Console in your browser.
2. Find your Fedora host in the Machines list.
3. Click the three-dot menu and select **Edit route settings**.
4. Toggle your subnet to **ON**.

## Step 3: Configure Firewalld Policies
Modern Fedora uses strict zone isolation. You need to assign the Tailscale interface to the `trusted` zone and create routing policies that explicitly allow traffic to pass between the Tailscale tunnel and the libvirt virtual network.

Run these commands on your Fedora host:

```bash
# Trust the Tailscale interface
sudo firewall-cmd --permanent --zone=trusted --add-interface=tailscale0

# Create policy to allow Tailscale -> Libvirt traffic
sudo firewall-cmd --permanent --new-policy tailscale-to-libvirt
sudo firewall-cmd --permanent --policy tailscale-to-libvirt --add-ingress-zone trusted
sudo firewall-cmd --permanent --policy tailscale-to-libvirt --add-egress-zone libvirt
sudo firewall-cmd --permanent --policy tailscale-to-libvirt --set-target ACCEPT

# Create policy to allow Libvirt -> Tailscale traffic
sudo firewall-cmd --permanent --new-policy libvirt-to-tailscale
sudo firewall-cmd --permanent --policy libvirt-to-tailscale --add-ingress-zone libvirt
sudo firewall-cmd --permanent --policy libvirt-to-tailscale --add-egress-zone trusted
sudo firewall-cmd --permanent --policy libvirt-to-tailscale --set-target ACCEPT

# Apply the changes
sudo firewall-cmd --reload
```

## Step 4: The Hidden Boss (Libvirt's nftables)
At this point, you might expect everything to work. However, pings to the VM will likely still fail. 

The culprit? `libvirt`. On modern Fedora, libvirt bypasses `firewalld` entirely and manages its own independent `nftables` chain called `libvirt_network`. This chain strictly rejects any forwarding traffic that wasn't initiated by the VM itself, completely ignoring the `firewalld` policies we just created.

To bypass this, we need to inject an `accept` rule directly into libvirt's private firewall table.

## Step 5: Automate the Fix with a Libvirt Hook
Because libvirt wipes and rebuilds its firewall rules every time the network starts or the host reboots, a manual `nftables` command will disappear. To make the fix permanent, create a libvirt hook script that re-injects the rule automatically.

1. Create the hooks directory and ensure the permissions are correct:

```bash
sudo mkdir -p /etc/libvirt/hooks
sudo chmod 755 /etc/libvirt/hooks
```

2. Open a new network hook script in your text editor:

```bash
sudo nano /etc/libvirt/hooks/network
```

3. Paste in this bash script. *(Note: You must use the absolute path `/usr/sbin/nft` because libvirt strips the system path for security before executing hooks)*:

```bash
#!/bin/bash

# Only trigger when the 'default' network is started
if [ "${1}" = "default" ] && [ "${2}" = "started" ]; then
    # Inject the rule into libvirt's private firewall table to allow Tailscale
    /usr/sbin/nft insert rule ip libvirt_network guest_input iifname "tailscale0" oifname "virbr0" accept
fi
```
*(Assuming `virbr0` is your default libvirt bridge).*

4. Save the file, make it executable, and restore the correct SELinux contexts (required on Fedora for the daemon to permit execution):

```bash
sudo chmod +x /etc/libvirt/hooks/network
sudo restorecon -R -v /etc/libvirt/hooks
```

5. Restart the libvirt daemons so they pick up the new SELinux states:

```bash
sudo systemctl restart virtnetworkd libvirtd
```

6. Start your virtual network:

```bash
sudo virsh net-start default
```

Your Fedora host will now dynamically maintain the correct `nftables` overrides alongside your `firewalld` policies, establishing a seamless and secure Tailscale connection directly to your virtual machines.
