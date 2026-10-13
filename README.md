# NGINX for a MERN Stack

A production-shaped `nginx.conf` that serves a React build as static files and reverse
proxies `/api/` to a pool of Node.js backends with load balancing, WebSocket support, and
forwarded client headers.

---

## Table of Contents

- [What This Config Does](#what-this-config-does)
- [Architecture](#architecture)
- [Extending This Setup](#extending-this-setup)
- [Installing](#installing)
- [Adapting It](#adapting-it)
- [Configuration Walkthrough](#configuration-walkthrough)
- [Verifying](#verifying)

---

## What This Config Does

- **Serves the React build** from `/var/www/frontend`, with SPA fallback so client-side
  routes work on a hard refresh
- **Load balances the API** across three backend instances using `least_conn`
- **Supports WebSockets** on the proxied API via connection upgrade headers
- **Preserves client information** through `X-Real-IP`, `X-Forwarded-For`, and
  `X-Forwarded-Proto`
- **Enables gzip**, `sendfile`, `tcp_nopush`, and keepalive for throughput
- **Quiets the noise** — `favicon.ico` and `robots.txt` are excluded from logs

---

## Architecture

One NGINX process serving two completely different kinds of traffic from the same port:
static files for the application shell, and proxied requests for the API.

```
                        :80
                         │
              ┌──────────┴──────────┐
              │   server block      │
              │   example.com       │
              └──────────┬──────────┘
                         │
        ┌────────────────┴────────────────┐
        │                                 │
  location /                        location /api/
  try_files $uri /index.html        proxy_pass
        │                                 │
        ▼                                 ▼
  /var/www/frontend              upstream backend_servers
  React build                    ┌──────────────────────┐
  (served from disk)             │  least_conn          │
                                 ├──────────────────────┤
                                 │ 127.0.0.1:5000       │
                                 │ 127.0.0.1:5001       │
                                 │ 127.0.0.1:5002       │
                                 └──────────────────────┘
```

### The two paths

| Path | Mechanism | Why |
| :--- | :--- | :--- |
| `/` and any unmatched URL | `try_files $uri /index.html` | Serves a real file if it exists, otherwise hands the SPA shell to the client so React Router can resolve the route |
| `/api/` | `proxy_pass http://backend_servers` | Forwards to the Node pool, load balanced |

`try_files` is what makes a hard refresh on `/dashboard` work. Without it NGINX would look
for a directory of that name, find nothing, and return 404 — the client-side router never
gets the chance to run.

### Load balancing

`least_conn` routes each request to the backend with the fewest active connections. That
suits an API where request durations vary, since round-robin can queue a slow request
behind another slow one on the same worker while a free worker sits idle.

### Proxy header contract

Once traffic passes through a proxy, the backend loses sight of the original client. Four
headers restore it, and the upgrade pair enables WebSockets:

| Header | Restores |
| :--- | :--- |
| `Host` | The hostname the client actually requested |
| `X-Real-IP` | The client address, for logging and rate limiting |
| `X-Forwarded-For` | The full proxy chain |
| `X-Forwarded-Proto` | Whether the original request was HTTPS |
| `Upgrade` / `Connection` | The HTTP→WebSocket handshake |

`proxy_http_version 1.1` is required alongside the upgrade headers — HTTP/1.0 has no
concept of a connection upgrade, so omitting that line breaks WebSockets silently while
ordinary requests keep working.

### Timeouts

`proxy_connect_timeout 5s` fails fast when a backend is down, so a dead worker costs five
seconds rather than hanging the request. `proxy_read_timeout 30s` bounds how long NGINX
waits for a response — note it also caps idle WebSocket connections, so long-lived sockets
need it raised.

## Extending This Setup

- **TLS** — `certbot --nginx -d yourdomain.com` rewrites the server block for HTTPS.
- **Pipeline** — a workflow validating this file with `nginx -t` on every pull request,
  linting it with `gixy`, and reloading the host after a successful check on merge.
- **Rate limiting** — `limit_req_zone` in the `http` block, applied to `/api/`.
- **Caching** — `proxy_cache` for idempotent API responses.

## Installing

Copy the config into place and reload:

```bash
# Linux
sudo cp "NGINX conf" /etc/nginx/nginx.conf
sudo nginx -t && sudo systemctl reload nginx

# macOS (Homebrew)
cp "NGINX conf" /opt/homebrew/etc/nginx/nginx.conf
nginx -t && brew services reload nginx
```

Always run `nginx -t` before reloading — it validates the syntax and refuses to apply a
broken config.

---

## Adapting It

Before using this on a real host, change the following:

| Setting | Current value | Change to |
|---|---|---|
| `server_name` | `example.com www.example.com` | Your domain |
| `root` | `/var/www/frontend` | Where your React build lives |
| `upstream backend_servers` | `127.0.0.1:5000-5002` | Your actual backend hosts/ports |
| `access_log` / `error_log` in the server block | `/opt/homebrew/var/log/nginx/` | Your platform's log path |

> The `http` block and the `server` block currently log to **different** paths — the
> Homebrew paths in the server block are macOS-specific. Make them consistent for your
> platform.

### Running fewer than three backends

The `upstream` block lists three servers. If you run only one, reduce it — NGINX will
otherwise mark the missing ones as failed on every request cycle:

```nginx
upstream backend_servers {
    least_conn;
    server 127.0.0.1:5000;
}
```

### Adding HTTPS

This config listens on port 80 only. For TLS, the simplest route is Certbot, which will
rewrite the server block for you:

```bash
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

---

## Configuration Walkthrough

**Load balancing method.** `least_conn` sends each request to the backend with the fewest
active connections, which suits requests of uneven duration better than round-robin. The
alternatives are plain round-robin (the default, omit the directive) and `ip_hash` for
sticky sessions.

**WebSocket upgrade.** `proxy_http_version 1.1` plus the `Upgrade` and `Connection`
headers are all three required — HTTP/1.0 cannot carry an upgrade, so omitting the version
line silently breaks WebSockets.

**Timeouts.** `proxy_connect_timeout 5s` fails fast when a backend is down;
`proxy_send_timeout` and `proxy_read_timeout` are 30s. Raise the read timeout if you have
long-polling or slow endpoints, since it also caps idle WebSocket connections.

**`default_type application/octet-stream`.** Files with an unrecognised extension download
rather than render — a safe default, relying on `mime.types` for everything known.

---

## Verifying

```bash
# Syntax check
nginx -t

# Confirm the SPA fallback works on a deep link
curl -I http://localhost/dashboard

# Confirm the API proxies through
curl -i http://localhost/api/health

# Watch requests distribute across backends
tail -f /var/log/nginx/access.log
```

---
