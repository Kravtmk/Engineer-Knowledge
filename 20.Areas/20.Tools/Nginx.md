## Check

```bash
nginx -v
```

```bash
systemctl status nginx
```

```bash
sudo nginx -t
```

## Control

```bash
sudo systemctl start nginx
```

```bash
sudo systemctl stop nginx
```

```bash
sudo systemctl restart nginx
```

```bash
sudo systemctl reload nginx
```

## Configuration

```bash
sudo vim /etc/nginx/nginx.conf
```

```bash
ls -la /etc/nginx/sites-available/
```

```bash
ls -la /etc/nginx/sites-enabled/
```

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/myapp \
           /etc/nginx/sites-enabled/myapp
```

Disable site:

```bash
sudo rm /etc/nginx/sites-enabled/myapp
```

Test before reload:

```bash
sudo nginx -t
```

```bash
sudo systemctl reload nginx
```

## Reverse Proxy

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
    }
}
```

## Logs

```bash
sudo tail -f /var/log/nginx/access.log
```

```bash
sudo tail -f /var/log/nginx/error.log
```

```bash
journalctl -u nginx
```

```bash
journalctl -u nginx -f
```

## Important Paths

```text
/etc/nginx/nginx.conf
/etc/nginx/sites-available/
/etc/nginx/sites-enabled/
/var/log/nginx/access.log
/var/log/nginx/error.log
```

## Quick Diagnosis

```bash
sudo nginx -t
systemctl status nginx
journalctl -u nginx
ss -lntp
curl http://localhost
```