# Tailscale — MIN

## Check

```bash
tailscale status
tailscale ip -4
systemctl status tailscaled
```

## Connect

```bash
sudo tailscale up
```

## Routes

Accept routes:

```bash
sudo tailscale set --accept-routes=true
```

Advertise LAN:

```bash
sudo tailscale set --advertise-routes=192.168.8.0/24
```

## Subnet Router

```text
Remote Client
    ↓
Tailscale
    ↓
Subnet Router
    ↓
Home LAN
```

Check forwarding:

```bash
sysctl net.ipv4.ip_forward
```

## Troubleshooting

```bash
tailscale status
systemctl status tailscaled
journalctl -u tailscaled -n 50
ls -l /dev/net/tun
```

## Remember

```text
Subnet Router = access to LAN
Exit Node     = Internet through remote network
```