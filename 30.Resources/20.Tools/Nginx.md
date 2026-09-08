# Nginx

## Overview

Nginx is a web server and reverse proxy.

A common DevOps setup:

```text
Client
  ↓
Nginx :80 / :443
  ↓
Application :8000
```

The application does not need to be directly exposed to the Internet.

For example:

```text
Internet
   ↓
Nginx :80
   ↓
127.0.0.1:8000
   ↓
Python App
```

---

## Web Server

Nginx can serve static files directly.

Example:

```nginx
server {
    listen 80;

    root /var/www/html;
    index index.html;
}
```

Flow:

```text
Browser
   ↓
Nginx
   ↓
/var/www/html/index.html
```

---

## Reverse Proxy

Nginx can receive a request and forward it to another application.

Example:

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
    }
}
```

Flow:

```text
Browser
   ↓
example.com:80
   ↓
Nginx
   ↓
127.0.0.1:8000
   ↓
Application
```

This is common for Python, Node.js and other backend applications.

---

## Configuration Structure

Main configuration:

```text
/etc/nginx/nginx.conf
```

On Debian/Ubuntu, site configurations commonly use:

```text
/etc/nginx/sites-available/
/etc/nginx/sites-enabled/
```

Example:

```text
sites-available/
└── myapp

sites-enabled/
└── myapp -> /etc/nginx/sites-available/myapp
```

`sites-available` contains configurations.

`sites-enabled` determines which site configurations are enabled.

A symbolic link commonly connects them.

---

## server Block

A `server` block describes how Nginx handles incoming requests.

```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
    }
}
```

Important directives:

```text
listen        → port Nginx listens on
server_name   → domain/hostname
location      → which requests this rule handles
proxy_pass    → where Nginx forwards requests
```

---

## Test Configuration

Before applying configuration changes:

```bash
sudo nginx -t
```

Successful result typically contains:

```text
syntax is ok
test is successful
```

This is an important operational habit:

```text
Edit
 ↓
nginx -t
 ↓
Reload
```

Do not blindly restart Nginx after editing configuration.

---

## Reload vs Restart

Reload:

```bash
sudo systemctl reload nginx
```

Nginx reloads its configuration without a normal full service stop/start cycle.

Restart:

```bash
sudo systemctl restart nginx
```

The service is stopped and started again.

For configuration changes, when possible:

```text
nginx -t
   ↓
reload
```

is preferable.

---

## Logs

Access log:

```text
/var/log/nginx/access.log
```

Records incoming requests.

Error log:

```text
/var/log/nginx/error.log
```

Records Nginx errors.

Systemd logs can also be inspected:

```bash
journalctl -u nginx
```

---

## Ports

Typical ports:

```text
80   HTTP
443  HTTPS
```

Check listening ports:

```bash
ss -lntp
```

Example architecture:

```text
0.0.0.0:80        Nginx
127.0.0.1:8000    Application
```

Nginx is exposed to clients while the application can remain bound to localhost.

---

## Common Troubleshooting Flow

If the website does not work:

```text
Is Nginx running?
        ↓
Is the config valid?
        ↓
Is Nginx listening?
        ↓
Can Nginx reach the application?
        ↓
What do the logs say?
```

Useful checks:

```bash
systemctl status nginx
sudo nginx -t
ss -lntp
curl http://localhost
curl http://127.0.0.1:8000
journalctl -u nginx
```

This helps determine whether the problem is:

```text
Nginx
Network/port
Configuration
Application
```

---

## Key Concept

Nginx and the application are different processes.

```text
Nginx running ≠ Application running

Application running ≠ Nginx configured correctly
```

Test each layer separately.