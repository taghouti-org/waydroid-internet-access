# Waydroid Custom Networking (No Host DNS Port 53 Conflict)

## ❓ Why This Fork/Modification?

Waydroid’s default `waydroid-net.sh` launches a `dnsmasq` server that listens on host port 53 for DNS, which conflicts with other DNS servers (AdGuard Home, Pi-hole, etc.) you might have running locally.

**This patched script disables DNS on host port 53 and serves only DHCP, letting you specify any DNS server for your Waydroid container.**

---

## 🛠️ What Has Changed?

- **dnsmasq DNS service is disabled with `--port=0`**  
  No process listening on host port 53 anymore.
- **Container DNS is assigned via DHCP**  
  With `--dhcp-option=6,8.8.8.8,8.8.4.4` (change as needed).
- **Firewall configuration updated**  
  - iptables/nftables: Only DHCP and forwarding rules are added.
  - No rules added for DNS (port 53).
- **Container gets NAT and DHCP as before; only DNS behaviour is altered.**
- **You may need to add custom nftables (or iptables) rules to allow traffic.**

---

## 💾 Installation

1. **Copy the patched script:**
    ```sh
    sudo cp /usr/lib/waydroid/data/scripts/waydroid-net.sh /usr/lib/waydroid/data/scripts/waydroid-net.sh.bak
    sudo cp waydroid-net.sh /usr/lib/waydroid/data/scripts/waydroid-net.sh
    sudo chmod +x /usr/lib/waydroid/data/scripts/waydroid-net.sh
    ```

2. **Restart Waydroid:**
    ```sh
    sudo systemctl restart waydroid-container
    ```

3. **Check port 53:**
    ```sh
    sudo ss -tulpn | grep :53
    # Only your own DNS server, not dnsmasq/waydroid, should appear.
    ```

---

## 🔥 Networking and Firewall Steps

The script will automatically add basic iptables/nftables rules for the LXC bridge.  
**However, depending on your distribution’s firewall policies, you may need to run these manual commands as well to ensure forwarding works:**

### If using nftables (“nft” mode):

```sh
sudo nft add rule ip filter FORWARD ip saddr 192.168.240.0/24 accept
sudo nft add rule ip filter FORWARD ip daddr 192.168.240.0/24 accept
```

- These rules allow traffic FROM and TO your Waydroid container subnet (`192.168.240.0/24`) to be forwarded through the main network interface.
- Make sure your `filter` table and `FORWARD` chain exist (most distros have these by default).

### If using iptables (classic mode):

```sh
sudo iptables -A FORWARD -i waydroid0 -j ACCEPT
sudo iptables -A FORWARD -o waydroid0 -j ACCEPT
```

- These rules do the same job for iptables-based firewalls.

---

### Ports/Ranges to ALLOW

| Protocol | Port | Interface      | Purpose    |
|----------|------|---------------|------------|
| UDP      | 67   | waydroid0     | DHCP Server|
| TCP      | 67   | waydroid0     | DHCP Server|
| FORWARD  | —    | waydroid0     | Container networking |
| NAT      | —    | waydroid0     | Outbound internet from container |

**No rules for port 53 are needed/added. Your own DNS server on the host uses the normal firewall setup.**

---

## 🔄 Example Full nftables Waydroid Section

```sh
# Allow DHCP to waydroid guests
nft add rule ip filter INPUT iif "waydroid0" udp dport 67 accept
nft add rule ip filter INPUT iif "waydroid0" tcp dport 67 accept

# Allow forwarding to/from waydroid subnet
nft add rule ip filter FORWARD ip saddr 192.168.240.0/24 accept
nft add rule ip filter FORWARD ip daddr 192.168.240.0/24 accept

# (NAT/postrouting usually handled by the script as MASQUERADE)
```

---

## 🧑‍💻 Setting Your Container’s DNS

In `waydroid-net.sh` you’ll find (edit to suit):

```sh
--dhcp-option=6,8.8.8.8,8.8.4.4  # Google DNS
# Or, to use your local server:
--dhcp-option=6,192.168.1.2      # Replace with your DNS IP
```

---

## ✅ Testing

- Boot Waydroid, check it gets an IP in `192.168.240.x`.
- Check DNS on Android (e.g. with `waydroid shell getprop | grep dns`).
- Confirm internet access inside the container.
- Confirm your own DNS server runs with nothing else on port 53.

---

## 🎯 Summary

- This setup stops Waydroid from hogging port 53 so you can use any local DNS server.
- Provides clear, minimal firewall instructions for both nftables and iptables setups.

**Enjoy seamless Android and DNS coexistence!**