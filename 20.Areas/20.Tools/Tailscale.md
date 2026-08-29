### Roadmap: Remote access → Home Lab


```
Architecture: 

T470 
 ↓ 
Internet / Mobile Hotspot 
 ↓ 
Tailscale 
 ↓ 
LXC 106 — tailscale 192.168.8.212
 ↓ 
Home LAN — 192.168.8.0/24
  ├── node0 — 192.168.8.200
  ├── node1 — 192.168.8.201
  ├── node2 — 192.168.8.202 
  └── other VMs 
```


| Step | Task                                        | Time   |
| ---- | ------------------------------------------- | ------ |
| 1    | Create  Tailscale account + install on T470 | 15 min |
| 2    | Create Linux VM/LXC в Proxmox               | 10 min |
| 3    | Install on LXC Tailscale                    | 15 min |
| 4    | Enable IP forwarding                        | 10 min |
| 5    | Create **subnet router** for home LAN       | 10 min |
| 6    | Approve the route Tailscale                 | 10 min |
| 7    | T470 → Oppo hotspot → Tailscale             | 10 min |
| 8    | Check`ping` / SSH to VM                     | 10 min |
| 9    | Check K3s: `kubectl get nodes`              | 10 min |

# 1. Install Tailscale on the Client

Client:
- ThinkPad T470
- Fedora

```
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

- Tailscale installed and authenticated

The T470 acts as the remote management workstation.

# 2. Create a Dedicated Tailscale LXC
# 2.1 Check the IP address


```
ping -c 3 192.168.8.211
```
```
ip neigh show 192.168.8.211
```
# 2. 2 LXC configuration

```
CT ID:       106
Hostname:    tailscale
OS:          Ubuntu 24.04
Unprivileged: yes
Nesting:     yes

CPU:         1 core
RAM:         512 MB
Swap:        512 MB
Disk:        8 GB

IP:          192.168.8.212/24
Gateway:     192.168.8.1
Bridge:      vmbr0
Firewall:    yes
```
# ## 2.3 Verify networking

Inside the container:

```
ip -br a

ping -c 3 192.168.8.1
ping -c 3 1.1.1.1
```
# 3. Install Tailscale

```
apt update
apt install curl
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```


# 4. Allow TUN Device Inside Proxmox LXC

## Problem

`tailscaled` may fail with:

```
CreateTUN("tailscale0") failed
/dev/net/tun does not exist
```

An LXC container uses the Proxmox host kernel and does not automatically  
have access to `/dev/net/tun`.

## Fix

On the Proxmox host:

```
ls -l /dev/net/tun
```

Expected device:

```
crw-rw-rw- 1 root root 10, 200 ... /dev/net/tun
```

Stop the container:

```
pct stop 106
```

Edit:

```
nano /etc/pve/lxc/106.conf
```

Add:

```
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
```

Start the container:

```
pct start 106
```

Inside the container verify:

```
ls -l /dev/net/tun
```

Then:

```
systemctl restart tailscaled
systemctl status tailscaled --no-pager
```

Expected:

```
Active: active (running)
```

# 5. Authenticate the Subnet Router

Inside the LXC:

```
tailscale up
```

Open the generated authentication URL and authorize the machine.

Verify:

```
tailscale status
tailscale ip -4
```

# 6. Enable IP Forwarding

The LXC must be allowed to route packets between the Tailscale network  
and the Home LAN.

Check:

```
sysctl net.ipv4.ip_forward
```

Enable immediately:

```
sysctl -w net.ipv4.ip_forward=1
```

Make it persistent:

```
cat >/etc/sysctl.d/99-tailscale.conf <<'EOF'
net.ipv4.ip_forward = 1
EOF
```

Apply:

```
sysctl --system
```

Verify:

```
sysctl net.ipv4.ip_forward
```

Expected:

```
net.ipv4.ip_forward = 1
```

# 7. Configure the Subnet Router

Advertise the Home LAN through Tailscale:

```
tailscale set --advertise-routes=192.168.8.0/24
```

This means:

> This Tailscale machine can route traffic to `192.168.8.0/24`.

Then open the Tailscale Admin Console:

Machine → tailscale → Edit route settings

Approve:

```
192.168.8.0/24
```

Do NOT enable Exit Node for this use case.

Subnet Router != Exit Node.

Subnet Router:  
remote client → Home LAN

Exit Node:  
remote client → all Internet traffic through Home Lab

# 8. Configure the T470 Client

On Fedora:

```
sudo tailscale set --accept-routes=true
```

Check:

```
tailscale status
```

The T470 can now use the route advertised by the Home Lab subnet router.

# 9. End-to-End Test

Important: test from OUTSIDE the Home LAN.

Example:

```
T470
 ↓
Oppo mobile hotspot
 ↓
Internet
 ↓
Tailscale
 ↓
LXC 106
 ↓
192.168.8.0/24
```

Test K3s node:

```
ping -c 3 192.168.8.200
```

Then:

```
ssh node0
```

Successful result:

```
IPv4 address for enp6s18: 192.168.8.200
```

This proves that remote access through the subnet router works.

# 10. Test K3s Management

After connecting to `node0`:

```
kubectl get nodes
```

Expected:

```
node0
node1
node2
```

This proves that the normal DevOps workflow can be performed remotely.

# 11. Start the Tailscale LXC Automatically

On Proxmox:

```
pct set 106 -onboot 1
```

Verify:

```
pct config 106 | grep onboot
```

Expected:

```
onboot: 1
```

# Troubleshooting

## tailscaled is not running

```
systemctl status tailscaled --no-pager
journalctl -u tailscaled -n 50 --no-pager
```

## TUN device is missing

```
ls -l /dev/net/tun
```

Check `/etc/pve/lxc/106.conf`.

## Home LAN is unreachable

Check forwarding:

```
sysctl net.ipv4.ip_forward
```

Check advertised routes:

```
tailscale status
```

Check that `192.168.8.0/24` is approved in the Tailscale Admin Console.

On the Linux client check:

```
tailscale status
```

and ensure route acceptance is enabled.

# Security Notes

- No SSH port is exposed directly to the public Internet.
- Tailscale provides an encrypted overlay network.
- The subnet router provides access to the Home LAN.
- The LXC is dedicated to remote access.
- The LXC runs as an unprivileged container.
- Exit Node is not enabled.
- Access can later be restricted using Tailscale ACLs / grants.

# Result

Remote Home Lab access:

```
T470
  ↓
Mobile Internet
  ↓
Tailscale
  ↓
LXC 106 — Subnet Router
  ↓
192.168.8.0/24
  ↓
VMs / K3s
```

Normal workflow remains unchanged:

```
ssh node0
ssh vm5
```

Status: ✅ WORKING
