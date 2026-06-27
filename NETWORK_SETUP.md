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
