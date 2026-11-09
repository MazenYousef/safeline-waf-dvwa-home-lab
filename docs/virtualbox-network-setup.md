# VirtualBox Network Setup: Kali and Ubuntu VMs

Both VMs need two network adapters, so that they can:

1. **reach the internet** to install packages and SafeLine, and
2. **talk to each other and to the Windows host** on a private network that carries all lab traffic.

[← Back to the main README](../README.md)

## Network design

| Machine       | Adapter 1: NAT (internet) | Adapter 2: Host-Only (lab) |
|---------------|---------------------------|----------------------------|
| Windows host  | n/a                       | 192.168.56.1               |
| Kali VM       | `eth0`, DHCP 10.0.2.x     | 192.168.56.106             |
| Ubuntu VM     | `enp0s3`, DHCP 10.0.2.x   | 192.168.56.104             |

**Why two adapters?** With NAT, a VM can reach the internet, but the other machines can't reach it. With Host-Only, the VMs and the host can reach each other, but nothing has internet. Using both gives internet access for installing software, plus an isolated network for the attack traffic. The vulnerable DVWA application is never exposed outside the host.

## 1. Configure the adapters in VirtualBox

For **each VM**, with the VM powered off, open **Settings > Network**:

- **Adapter 1:** set *Attached to* to **NAT**.
- **Adapter 2:** set *Attached to* to **Host-Only Adapter**, and select *VirtualBox Host-Only Ethernet Adapter*.

## 2. Check the IP addresses

On the Windows host:

```powershell
ipconfig     # VirtualBox Host-Only Network -> 192.168.56.1
```

On each VM:

```bash
ip -br addr
```

## 3. Fix the NAT connection on Kali (if needed)

If the NAT adapter has no IP address, Kali has no internet. Check the connections and devices:

```bash
nmcli connection show
nmcli device status
```

If `eth0` has no active DHCP connection, create one and bring it up:

```bash
sudo nmcli connection add type ethernet ifname eth0 con-name NAT-Internet ipv4.method auto connection.autoconnect yes
sudo nmcli connection up NAT-Internet
```

`connection.autoconnect yes` makes the connection come back automatically after a reboot.

## 4. Test connectivity

```bash
# From Kali
ping -c 3 8.8.8.8            # internet via NAT
ping -c 3 192.168.56.104     # Ubuntu web server via Host-Only
```

## Ports used in this lab

| Port      | On             | Purpose                                         |
|-----------|----------------|-------------------------------------------------|
| TCP 443   | Ubuntu (SafeLine) | HTTPS entry point for the application         |
| TCP 80    | Ubuntu (SafeLine) | HTTP, disabled for the application in this lab |
| TCP 9443  | Ubuntu (SafeLine) | SafeLine management dashboard                 |
| TCP 8080  | Ubuntu (Apache)   | Backend web server, upstream for SafeLine     |
| TCP 3306  | Ubuntu (MySQL)    | Database, used locally by DVWA only           |
