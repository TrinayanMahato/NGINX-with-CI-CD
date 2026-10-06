# NGINX for a MERN Stack

A production-shaped `nginx.conf` that serves a React build as static files and reverse
proxies `/api/` to a pool of Node.js backends with load balancing, WebSocket support, and
forwarded client headers.

---

## Table of Contents

- [What This Config Does](#what-this-config-does)
- [Request Routing](#request-routing)
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

## Request Routing

```
                    ┌─────────────────────────────┐
  GET /            │  try_files $uri /index.html │ ──► /var/www/frontend
  GET /dashboard   │  (SPA fallback)             │
                    └─────────────────────────────┘

                    ┌─────────────────────────────┐      127.0.0.1:5000
  GET  /api/users   │  proxy_pass                 │ ──►  127.0.0.1:5001
  POST /api/login   │  http://backend_servers     │      127.0.0.1:5002
  WS   /api/socket  │  (least_conn)               │
                    └─────────────────────────────┘
```

Any path that is not a real file falls through to `index.html`, which is what lets React
Router handle `/dashboard` without a 404 from NGINX.

---

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

## On the CI/CD in the Name

The repository is named *NGINX-with-CI-CD*, but it currently contains only the NGINX
configuration — there is no pipeline definition here yet. A natural next step is a
workflow that validates and ships this file, for example:

- [ ] Run `nginx -t` against the config on every pull request
- [ ] Lint with `gixy` for common misconfigurations and security issues
- [ ] Deploy to the target host on merge to `main`
- [ ] Reload NGINX only after the syntax check passes on the host
