# Nginx — Practice

## Level 1 — Install & Inspect

On a test VM:

-  Install Nginx.
    
-  Check the installed version.
    
-  Check whether the service is running.
    
-  Find the main Nginx configuration.
    
-  Find the access log.
    
-  Find the error log.
    
-  Determine which port Nginx is listening on.
    

Do not use `Areas/Nginx.md`.

---

## Level 2 — Web Server

Create a simple page:

```html
<h1>My first Nginx site</h1>
```

Then:

-  Serve it through Nginx.
    
-  Open it using `curl`.
    
-  Find the request in the access log.
    
-  Verify the Nginx configuration before applying changes.
    

Expected:

```text
curl
 ↓
Nginx :80
 ↓
HTML
```

---

## Level 3 — Break & Fix

Intentionally introduce an error into a test Nginx configuration.

Then:

-  Test the configuration.
    
-  Read the error.
    
-  Find the broken configuration.
    
-  Fix it.
    
-  Test again.
    
-  Apply the corrected configuration.
    

### Question

Why should you run a configuration test before reload/restart?

---

## Level 4 — Reverse Proxy

Run an application locally on:

```text
127.0.0.1:8000
```

Configure Nginx so that:

```text
curl http://localhost
        ↓
      Nginx
        ↓
127.0.0.1:8000
        ↓
       App
```

Tasks:

-  Verify the application directly on port `8000`.
    
-  Create an Nginx configuration.
    
-  Configure reverse proxying.
    
-  Test the Nginx configuration.
    
-  Reload Nginx.
    
-  Access the application through Nginx on port `80`.
    

---

## Level 5 — Troubleshooting

Scenario:

```text
curl http://127.0.0.1:8000
```

works.

But:

```text
curl http://localhost
```

does not.

Without changing anything first:

-  Check Nginx status.
    
-  Check configuration syntax.
    
-  Check listening ports.
    
-  Check Nginx logs.
    
-  Determine whether the problem is Nginx or the application.
    

---

## Level 6 — Junior DevOps Incident

You deploy:

```text
Python App :8000
      ↑
    Nginx :80
```

Users report:

```text
502 Bad Gateway
```

Investigate without immediately restarting everything.

Determine:

-  Is Nginx running?
    
-  Is the application running?
    
-  Is port `8000` listening?
    
-  Can you reach the application directly?
    
-  Is `proxy_pass` pointing to the correct address and port?
    
-  What does the Nginx error log say?
    

### Question

If Nginx is `active (running)`, can the website still be broken?

Explain why.

---

## Final Challenge — Simple Deploy

Deploy a small application behind Nginx.

Final architecture:

```text
Client
  ↓
Nginx :80
  ↓
Application :8000
```

Requirements:

-  Application runs as a service.
    
-  Nginx acts as reverse proxy.
    
-  Application port is verified independently.
    
-  Nginx configuration passes validation.
    
-  Application works through Nginx.
    
-  Access and error logs can be inspected.
    
-  Restart the application and verify Nginx still routes to it.
    
-  Break the application and diagnose the failure from the outside in.
    

**Goal:** understand which layer is broken instead of randomly restarting services.