# Network Setup Guide — Dual NIC Configuration

This lab uses two network adapters on both the controller and SAP server to separate SAP traffic from internet access (needed for collection installation).

---

## Adapter Roles

| Adapter | Type | Purpose | IP Range |
|---------|------|---------|----------|
| `ens160` | Bridge (Host-Only) | SAP internal network — Ansible SSH, SAP ports | 192.168.1.x |
| `ens224` | NAT | Internet access — package/collection downloads | DHCP (NAT) |

---

## Configuration Commands

Run on both `ansnode` and `saperp`:

```bash
# Show current adapters
nmcli con show

# Set NAT adapter as default route (for internet access)
nmcli con mod ens224 ipv4.method auto ipv4.never-default no

# Remove default route from bridge adapter (SAP-only traffic)
nmcli con mod ens160 ipv4.never-default yes

# Apply changes
nmcli con up ens224
nmcli con up ens160

# Verify default route is via NAT adapter
ip route
```

Expected output of `ip route`:
```
default via <NAT_GATEWAY> dev ens224 proto dhcp ...
192.168.1.0/24 dev ens160 proto kernel scope link ...
```

---

## Why Explicit `sap_ip` in Variables

Because `saperp` has multiple IP addresses (one per adapter), SAP software can bind to the wrong interface during installation. Setting `sap_ip: "192.168.1.17"` explicitly in `sap_common.yml` ensures all SAP roles bind to the correct bridge adapter IP.

```yaml
# inventory/group_vars/sap_servers/sap_common.yml
sap_ip: "192.168.1.17"       # bridge adapter - SAP internal network
sap_hostname: "saperp"
sap_fqdn: "saperp.tes.com"
```

---

## /etc/hosts Configuration

Required on the **controller** (`ansnode`):

```
192.168.1.18  ansnode.tes.com  ansnode
192.168.1.17  saperp.tes.com   saperp
```

SAP automation relies on hostname resolution. If `/etc/hosts` is incomplete, role tasks that check FQDN resolution will fail.

Verify with:
```bash
hostname          # should return: saperp
hostname -f       # should return: saperp.tes.com
ping saperp       # should resolve to 192.168.1.17
```

## NOTES

During the setup using two NIC, at first internet connection won't connect despite ping into 8.8.8.8 success, but when try to ping into google.com it's hang and don't show any reply back. But at that time, there is a clue why the connection were not established. The reply from ping test to google.com shows that it uses ipv6 instead of ipv4. Therefore I decided to disable all of the ipv6 service on both nodes, and retry again to ping google and use dnf repolist, which is succesfull. 

```
#Disable the ipv6 services
nmcli con mod <device_name> ipv6.method "disabled"
sysctl -w net.ipv6.conf.all.disable_ipv6=1
sysctl -w net.ipv6.conf.default.disable_ipv6=1

#restart network adapter
nmcli con down <device_name> && nmcli con up <device_name>
systemctl restart NetworkManager

#check /etc/resolv.conf
cat /etc/resolv.conf

```
