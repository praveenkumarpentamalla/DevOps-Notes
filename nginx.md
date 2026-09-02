# NGINX: Beginner → Expert → Production Mastery Syllabus

*A complete DevOps/Platform Engineering curriculum for Nginx: Design, Configure, Deploy, Secure, Optimize, Monitor, Troubleshoot, Operate.*

**Legend:** ⭐ Must Know | 🔥 Advanced | 🚀 Expert | 🏭 Production Critical

This syllabus assumes you already know Linux, Docker, Kubernetes, AWS, Terraform, Git/CI-CD, PostgreSQL, DNS, SSL/TLS basics, load balancers, and Nginx basics — so those fundamentals are referenced only where they intersect directly with Nginx behavior, not taught from scratch.

---

## How This Document Is Organized

Each **directive** you meet is explained as:

`Syntax → Default → Context → Purpose → Example → Production Considerations`

Each **topic** is explained as:

`What to learn → Why it matters → Key concepts → Directives → Practical example → Production use case → Common mistakes → Troubleshooting → Labs → Interview Qs`

Because of the enormous scope you asked for (27 parts, 15+ projects, 20+ incidents, 140+ interview questions, a 30-day plan), this document is organized so every part is genuinely useful as a *reference you keep coming back to*, not just a table of contents. Dense reference tables are used where repeating the full 6-part breakdown for every single directive would just be noise — the directives that actually change production behavior get full depth.

---

# PART 1 — NGINX FUNDAMENTALS ⭐

## 1.1 What Is Nginx, Really

Nginx is an **event-driven, asynchronous, non-blocking** web server / reverse proxy / load balancer / API gateway / mail proxy / TCP-UDP proxy, originally built by Igor Sysoev to solve the **C10K problem** (handling 10,000+ concurrent connections efficiently). Apache's traditional model spawns a thread or process per connection; Nginx instead uses a small, fixed number of **worker processes**, each running a non-blocking event loop that can juggle thousands of connections at once.

**Why this matters:** almost every Nginx performance and troubleshooting question ("why is CPU pegged," "why are connections queuing," "why did one slow backend request stall others") traces back to this architecture. If you understand the event loop, you can reason about behavior instead of memorizing directives.

### Nginx vs Apache (conceptual, not memorization)

| Dimension | Nginx | Apache (prefork/worker MPM) |
|---|---|---|
| Concurrency model | Event-driven, async, non-blocking | Process/thread per connection (traditional) |
| Memory per connection | Very low (event loop, shared workers) | Higher (thread/process overhead) |
| Static file serving | Extremely fast | Slower, more overhead |
| Dynamic content | Delegates via proxy_pass/FastCGI/uwsgi | Can embed interpreters (mod_php) |
| Config reload | Graceful, zero-downtime | Similar, but historically heavier |
| Typical role today | Reverse proxy / LB / edge / ingress | Less common at the edge in modern stacks |

### Core Roles Nginx Plays

- **Web server** — serves static files directly from disk.
- **Reverse proxy** — sits in front of app servers, forwards client requests to them, returns responses to clients. Clients only ever see Nginx.
- **Forward proxy** — sits in front of *clients*, forwards their requests outward (rare in typical web stacks, common in corporate egress filtering).
- **Load balancer** — distributes requests across multiple upstream servers.
- **API gateway (conceptually)** — routing, rate limiting, auth headers, TLS termination in front of microservices (full API gateway features need Nginx Plus or OpenResty/Lua for anything beyond basic routing).
- **Static file server** — CDNs and SPAs often use Nginx as the origin.
- **TCP/UDP proxy** — via the `stream` module (Part 20).

## 1.2 Process Architecture ⭐

```
                ┌────────────────────┐
                │   Master Process    │  (reads config, binds ports,
                │   (runs as root)    │   spawns/manages workers,
                └─────────┬───────────┘   handles signals)
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
  ┌───────────┐     ┌───────────┐     ┌───────────┐
  │ Worker #1 │     │ Worker #2 │     │ Worker #3 │   (runs as unprivileged
  │ event loop│     │ event loop│     │ event loop│    user, e.g. nginx/www-data)
  └───────────┘     └───────────┘     └───────────┘
        │                 │                 │
   thousands of      thousands of      thousands of
   connections        connections        connections
```

- **Master process**: owns privileged operations — reading `nginx.conf`, binding to ports <1024, spawning/killing workers, handling OS signals, log rotation coordination. Runs as root (or the configured user) so it can bind privileged ports and then workers drop privileges.
- **Worker processes**: do the actual work — accept connections, read requests, talk to upstreams, write responses. Each worker runs a single-threaded event loop (though some I/O like disk reads can use a thread pool via `aio threads`). Typically one worker per CPU core (`worker_processes auto;`).
- **Cache manager / cache loader** (extra processes when caching is enabled): manage on-disk cache expiry and loading cache metadata at startup.

### Event-Driven Model

Each worker uses OS-level event notification (`epoll` on Linux, `kqueue` on BSD) to know which of its thousands of open sockets are ready to read/write, instead of blocking on each one. This is why one worker can handle 10,000+ simultaneous connections with minimal memory: no per-connection thread stack.

**worker_connections** caps how many simultaneous connections *one worker* can hold open (including connections to upstreams, not just clients) — so max theoretical concurrent connections ≈ `worker_processes × worker_connections`, but real max client connections is roughly half that when Nginx proxies (each client connection may open a second connection to the upstream).

### Request Lifecycle (simplified)

1. Client TCP handshake → OS accepts socket → worker's event loop picks it up.
2. TLS handshake (if HTTPS) — worker performs it (CPU-bound work, why TLS termination affects CPU).
3. Worker reads HTTP request line + headers into buffers.
4. Nginx runs through configured **phases**: post-read → server rewrite → find-config (server_name match) → rewrite → pre-access → access (auth, IP allow/deny) → content generation (static file, proxy_pass, return, etc.) → log.
5. Response streamed back to client; connection kept alive or closed per `keepalive` settings.

## 1.3 Installation & Directory Structure ⭐

Typical Debian/Ubuntu layout (varies slightly by distro/package):

```
/etc/nginx/
├── nginx.conf              # main config, top-level context
├── conf.d/                 # extra config snippets, *.conf auto-included
│   └── default.conf
├── sites-available/        # all defined virtual hosts (not necessarily active)
│   └── example.com.conf
├── sites-enabled/          # symlinks into sites-available = "active" sites
│   └── example.com.conf -> ../sites-available/example.com.conf
├── modules-enabled/        # dynamic module loading
├── mime.types              # extension → Content-Type mapping
├── fastcgi_params, uwsgi_params, scgi_params
├── snippets/               # reusable config fragments (SSL params, etc.)
└── ssl/                    # (convention, not required) certs/keys

/var/log/nginx/
├── access.log
└── error.log

/var/www/html/              # default document root
/usr/sbin/nginx             # binary
```

RHEL/CentOS/Alpine differ (`/etc/nginx/conf.d/*.conf` is the common pattern instead of sites-available/enabled). **Know both** — Docker images (`nginx:alpine`) typically use `conf.d` only, no sites-available.

**Common mistake:** editing `sites-available` but forgetting to symlink into `sites-enabled` (or vice versa, forgetting to remove the symlink to disable a site) — site silently doesn't take effect or old config lingers.

### Lab 1
1. Install Nginx (`apt install nginx` or run `docker run -p 8080:80 nginx:alpine`).
2. Locate and read through `nginx.conf`, `mime.types`, and the default site config.
3. Run `nginx -t` and `nginx -T` (dump full merged config) and compare.
4. Create a new site in `sites-available`, symlink it into `sites-enabled`, reload, verify with `curl`.

### Interview Qs (Fundamentals)
- Explain Nginx's process model and why it scales better than a thread-per-connection server under high concurrency.
- What happens when a worker process crashes? (Master respawns it; existing connections on that worker drop.)
- Why is `worker_processes auto` usually correct, and when would you deviate?
- What's the difference between `nginx -s reload` and restarting the service?

---

# PART 2 — NGINX CONFIGURATION ⭐

## 2.1 Contexts & Inheritance

Nginx config is a **hierarchy of contexts** (blocks), each narrowing scope:

```
main (top-level, outside any block)
 └─ events { }
 └─ http { }
     └─ server { }
         └─ location { }
             └─ location { } (nested)
 └─ stream { }   (TCP/UDP, Part 20)
 └─ mail { }     (Part 20)
```

**Inheritance rule:** most directives set in a parent context are inherited by children *unless the child explicitly overrides them*. Some directives (like `add_header`, `proxy_set_header`) have a special quirk: **if you set even one instance of them in a child context, the ones from the parent are NOT inherited — you must repeat them.** This is one of the most common real-world bugs.

```nginx
http {
    add_header X-Frame-Options DENY;

    server {
        location /api/ {
            add_header X-Custom-Header "value";
            # X-Frame-Options is now LOST here — must be re-added explicitly!
        }
    }
}
```

## 2.2 Core Directives Reference (Deep Dive)

### `listen` ⭐
- **Syntax:** `listen address[:port] [default_server] [ssl] [http2] [backlog=N];`
- **Default:** `*:80` (or 8000 if run as non-root without explicit config)
- **Context:** `server`
- **Purpose:** binds the server block to an address/port and marks TLS/HTTP2/default behavior.
- **Example:** `listen 443 ssl http2;`
- **Production considerations:** `default_server` decides which server block answers requests that don't match any `server_name` (critical for preventing information leakage / certificate confusion attacks). Always explicitly set one default_server per port, ideally one that returns 444 or a generic response, not your primary app.

### `server_name` ⭐
- **Syntax:** `server_name name1 name2 ...;` (supports exact, wildcard `*.example.com`, and regex `~^(?<sub>.+)\.example\.com$`)
- **Default:** `""` (empty)
- **Context:** `server`
- **Purpose:** matches the `Host` header to select which server block handles the request.
- **Production considerations:** matching precedence: exact match → longest wildcard starting with `*` → longest wildcard ending with `*` → first matching regex, in configuration order. Always have a `default_server` catch-all to prevent host header attacks routing to the wrong app.

### `root` vs `alias` ⭐ (classic gotcha)
- `root /var/www/html;` inside `location /images/` → requested `/images/cat.png` resolves to `/var/www/html/images/cat.png` (path is **appended**).
- `alias /var/www/uploads/;` inside `location /images/` → requested `/images/cat.png` resolves to `/var/www/uploads/cat.png` (location prefix is **replaced**).
- **Common mistake:** using `alias` without a trailing slash, or expecting `alias` to append like `root` does — causes 404s that look inexplicable until you understand this distinction.

### `try_files` ⭐🏭
- **Syntax:** `try_files path1 path2 ... final;`
- **Context:** `server`, `location`
- **Purpose:** checks files/dirs in order, serving the first that exists; the final argument is usually a named location or `=404`.
- **SPA example:**
```nginx
location / {
    root /var/www/app;
    try_files $uri $uri/ /index.html;
}
```
This lets client-side routers (React Router, Vue Router) handle deep links: any path not matching a real file falls through to `index.html`.

### `proxy_pass` and friends ⭐🏭 (see Part 6 for full depth)
- **Syntax:** `proxy_pass http://backend;`
- **Critical trailing-slash rule:**
  - `location /api/ { proxy_pass http://backend/; }` → `/api/users` → `http://backend/users` (prefix stripped, replaced)
  - `location /api/ { proxy_pass http://backend; }` (no trailing slash on proxy_pass) → `/api/users` → `http://backend/api/users` (prefix preserved)
  - This single trailing slash is one of the most common sources of "why is my proxy 404ing" bugs in the world.

### `return` vs `rewrite`
- `return 301 https://$host$request_uri;` — immediate, efficient, no regex evaluation needed for simple redirects. **Prefer this over rewrite for redirects.**
- `rewrite ^/old/(.*)$ /new/$1 permanent;` — regex-capable, more powerful but more expensive (evaluated per request, can cause infinite loop bugs if misconfigured) — use only when you need pattern capture/substitution.

### `client_max_body_size` 🏭
- **Default:** `1m`
- **Context:** `http, server, location`
- **Purpose:** caps request body size; exceeding it returns `413 Request Entity Too Large`.
- **Production consideration:** must be raised for file upload endpoints — a *very* common production incident (Part 24, Incident #4).

### Performance/connection directives
`sendfile on;` (zero-copy file serving via kernel), `tcp_nopush on;` (batch packets before sending — pairs with sendfile), `tcp_nodelay on;` (disable Nagle's algorithm for low latency on keepalive connections), `keepalive_timeout 65;`, `keepalive_requests 1000;`. Covered in depth in Part 11.

### Logging directives
`log_format main '$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent" "$http_x_forwarded_for"';`
`access_log /var/log/nginx/access.log main;`
`error_log /var/log/nginx/error.log warn;` — levels: debug, info, notice, warn, error, crit, alert, emerg.

## 2.3 Testing, Reloading, Signals ⭐🏭

| Command | What it does |
|---|---|
| `nginx -t` | Test config syntax **without applying** |
| `nginx -T` | Test AND print the full merged configuration (all includes resolved) |
| `nginx -s reload` | Master re-reads config, spawns new workers with new config, gracefully drains and kills old workers — **zero downtime** |
| `nginx -s reopen` | Reopen log files (used after log rotation) |
| `nginx -s quit` | Graceful shutdown (finish in-flight requests) |
| `nginx -s stop` | Immediate/fast shutdown |
| `kill -HUP <master_pid>` | Same as reload |
| `kill -USR2 <master_pid>` | Binary upgrade (hot-swap the nginx binary itself with zero downtime) |

**Always run `nginx -t` before every reload in production/CI.** A bad config on reload does NOT take down a running Nginx (old workers keep serving) but *will* prevent a fresh start — so a bad config plus a full restart (not reload) is a classic self-inflicted outage.

### Lab 2
1. Deliberately break your config (typo a directive), run `nginx -t`, read the error, fix it.
2. Change a config value, reload, and use `curl -v` to confirm the change took effect without dropping an in-flight `curl` request (use `sleep` via a slow backend to prove zero-downtime reload).
3. Compare `nginx -T` output before/after adding an `include`.

### Interview Qs (Configuration)
- What's the difference between `root` and `alias`, and how does that affect path resolution?
- Explain why `proxy_pass` with vs without a trailing slash produces different upstream URLs.
- Why does `add_header` "disappear" when set in a nested location after being set in the parent?
- Walk through what happens, step by step, when you run `nginx -s reload`.

---

# PART 3 — STATIC WEB SERVER

## What to Learn
Serving static assets efficiently: MIME types, caching headers, compression, permissions, SPA routing.

## Why It Matters
Even in proxy-heavy architectures, Nginx usually still serves the SPA shell, images, and other static assets directly — and does it dramatically faster than any app server. Misconfigured caching here is one of the most common "why don't my users see the new deployment" bugs.

## Key Concepts & Directives
- `expires 30d;` / `add_header Cache-Control "public, immutable";` — long-cache hashed assets (e.g., `app.a1b2c3.js`), **never** long-cache `index.html`.
- `gzip on; gzip_types text/css application/javascript application/json image/svg+xml; gzip_min_length 256; gzip_comp_level 5;`
- `brotli on; brotli_types ...;` (requires the brotli module, not built in by default on all distros).
- ETag is on by default; disable (`etag off;`) only if you have a specific reason (e.g., using immutable hashed filenames exclusively).

## Production Example (SPA + static assets)
```nginx
server {
    listen 443 ssl http2;
    server_name app.example.com;

    root /var/www/app/dist;
    index index.html;

    # Hashed, versioned assets: cache aggressively
    location ~* \.(?:js|css|woff2?|png|jpg|jpeg|svg|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;             # reduce log noise for assets
    }

    # index.html must NEVER be cached — it's how users get new deploys
    location = /index.html {
        add_header Cache-Control "no-store";
    }

    # SPA fallback for client-side routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    gzip on;
    gzip_types text/plain text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;
}
```
**Line-by-line:** the regex location catches hashed static assets and marks them immutable for a year (safe because the filename changes on every new build); `index.html` is explicitly excluded from that rule and forced to `no-store` so browsers always fetch the latest shell, which then references the newly-hashed JS/CSS; the catch-all `location /` implements SPA client-side routing by falling back to `index.html` for any path that isn't a real file.

## Common Mistakes
- Caching `index.html` long-term → users stuck on old deploys until hard refresh.
- Forgetting `try_files ... /index.html` → deep-linking to a client-side route (e.g., `/dashboard/settings`) 404s on refresh.
- Serving symlinked directories without `disable_symlinks` consideration in security-sensitive setups.

## Troubleshooting
- **Symptom:** old JS/CSS served after deploy → check `Cache-Control`/`Expires` headers with `curl -I`, verify hashed filenames actually changed, check CDN cache in front of Nginx too.
- **Symptom:** direct URL to SPA route 404s → verify `try_files` fallback and that it's inside the correct `location /`.

## Lab
Deploy a React/Vue build behind Nginx with proper cache headers; verify with `curl -I` that hashed assets get `Cache-Control: public, immutable` and `index.html` gets `no-store`; test deep-link refresh works.

---

# PART 4 — SERVER BLOCKS & VIRTUAL HOSTING ⭐

## Key Concepts
Multiple domains/subdomains served by one Nginx instance, matched by `server_name` (Host header) and/or `listen` port. This is how one Nginx box fronts `api.example.com`, `app.example.com`, `admin.example.com`, and the bare `example.com` simultaneously.

```nginx
server {
    listen 443 ssl http2;
    server_name example.com www.example.com;
    return 301 https://app.example.com$request_uri;   # apex/marketing redirect
}

server {
    listen 443 ssl http2;
    server_name app.example.com;
    root /var/www/app;
    # ...
}

server {
    listen 443 ssl http2;
    server_name api.example.com;
    location / { proxy_pass http://api_upstream; }
}

server {
    listen 443 ssl http2;
    server_name admin.example.com;
    # extra auth/IP restriction here
    allow 10.0.0.0/8;
    deny all;
    location / { proxy_pass http://admin_upstream; }
}

# Catch-all default server — protects against Host header attacks
server {
    listen 443 ssl default_server;
    server_name _;
    ssl_certificate /etc/nginx/ssl/default.crt;   # a "junk" cert, not your real one
    return 444;   # close connection without response
}
```

### `server_name` Matching Precedence (memorize the order)
1. Exact name (`example.com`)
2. Longest wildcard starting with `*` (`*.example.com`)
3. Longest wildcard ending with `*` (`www.*`)
4. First matching regular expression, in the order defined in the config

## Common Mistakes
- No `default_server` defined → Nginx picks the *first* server block in file-processing order as default, which may leak the wrong app/cert to arbitrary Host headers.
- Duplicate `server_name` across two server blocks on the same port → unpredictable/ambiguous matching, `nginx -t` won't always catch this as an error.

## Troubleshooting
Use `curl -H "Host: fake.example.com" https://your-ip/` to test which server block actually answers for a given Host header — invaluable for debugging virtual host misrouting.

## Interview Qs
- How does Nginx decide which server block handles an incoming request when multiple blocks listen on the same port?
- Why is a `default_server` catch-all important for security?

---

# PART 5 — LOCATION MATCHING 🔥⭐

This is the topic that trips up even experienced engineers. Get this right and 80% of "why is my routing wrong" debugging disappears.

## Location Matching Types & Precedence (in actual evaluation order)

1. **Exact match** `location = /path` — highest priority, stops search immediately on match.
2. **Preferential prefix** `location ^~ /path` — if this is the longest matching prefix, regex locations are skipped entirely.
3. **Regex, case-sensitive** `location ~ \.php$`
4. **Regex, case-insensitive** `location ~* \.(jpg|png)$` — regex locations 3 & 4 are checked *in the order they appear in the config*, first match wins (NOT longest match).
5. **Prefix match (no modifier)** `location /path` — longest matching prefix wins among these, but only used as fallback if no regex matched.

**The actual algorithm:**
1. Nginx first finds the longest matching prefix location (regardless of `^~`).
2. If that longest-prefix match had `^~`, or if it was an `=` exact match, Nginx uses it immediately and stops.
3. Otherwise, Nginx then checks all `~`/`~*` regex locations *in config order* — first regex match wins, **even if a longer prefix match existed**.
4. If no regex matches, Nginx falls back to the longest matching prefix location found in step 1.

### Worked Example
```nginx
location / { ... }                      # A: prefix
location /images/ { ... }               # B: prefix (longer than A)
location ^~ /images/thumb/ { ... }      # C: preferential prefix (longer than B)
location ~* \.(jpg|jpeg|png)$ { ... }   # D: regex
```
Request `/images/thumb/cat.jpg`:
- Longest prefix match is C (`/images/thumb/`), and it has `^~` → **Nginx stops here, uses C**, even though D would also match and D is a regex. This is the entire *purpose* of `^~`: to say "if this prefix matches, don't even bother checking regexes."

Request `/images/cat.jpg` (not under `/thumb/`):
- Longest prefix match is B (`/images/`), no `^~` modifier → Nginx proceeds to check regexes → D matches → **D wins**, even though B is a valid, longer, more "specific-looking" match. This surprises people constantly.

## Common Mistakes
- Assuming prefix locations always lose to regex — they only lose if no `^~` is present on the best prefix match.
- Multiple regex locations that could both match — remember it's config **order**, not specificity.
- Forgetting nested locations inherit but can also independently override things like `proxy_pass` target, causing partial-config surprises.

## Debugging Technique
Add a temporary `add_header X-Matched-Location "name-here" always;` in each candidate location during testing, then `curl -I` to see which one actually fired. Also use `nginx -T` and `error_log ... debug;` (Part 15) for deep tracing.

## Lab
Build the 4-location example above, run curl tests against `/`, `/images/cat.jpg`, `/images/thumb/cat.jpg`, `/images/thumb/doc.pdf`, and predict the winner *before* checking — then verify.

## Interview Qs
- Explain, step by step, how Nginx picks a location block when both a prefix and a regex could match.
- What does `^~` actually do, precisely?
- Why might two regex locations both matching a URL produce a "wrong" result, and how do you fix it?

---

# PART 6 — REVERSE PROXY 🏭⭐

## Why Reverse Proxying Is Nginx's Signature Use Case
It terminates client connections (TLS, HTTP/1.1 quirks, slow clients) and speaks a clean, fast, often keepalive-pooled connection to your application servers — decoupling client-facing concerns from application concerns.

## The Header Rewriting Problem (critical to understand)
By default, when Nginx proxies a request, the backend sees a request that looks like it came **from Nginx**, not from the real client — because TCP connection info (source IP) is naturally Nginx's IP, and the `Host` header by default becomes whatever `proxy_pass` target is unless you explicitly forward it.

```nginx
upstream backend { server 10.0.1.10:8080; server 10.0.1.11:8080; }

server {
    listen 443 ssl http2;
    server_name api.example.com;

    location /api/ {
        proxy_pass http://backend/;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Connection "";        # clear for keepalive to upstream

        proxy_connect_timeout 5s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        proxy_buffering on;
        proxy_buffer_size   4k;
        proxy_buffers       8 8k;
    }
}
```

**Line-by-line:**
- `Host $host` — forwards the original Host header the client sent, so the backend app (which may itself do host-based routing, or generate absolute URLs) sees the real domain.
- `X-Real-IP $remote_addr` — the immediate client IP as Nginx sees it.
- `X-Forwarded-For $proxy_add_x_forwarded_for` — appends to any existing XFF chain (important behind multiple proxies/CDNs — always append, never just set `$remote_addr`, or you lose the chain from upstream proxies like a CDN).
- `X-Forwarded-Proto $scheme` — tells the backend whether the original request was HTTP or HTTPS (critical: without this, apps behind TLS-terminating Nginx often generate `http://` links/redirects incorrectly, or think all traffic is insecure).
- `proxy_http_version 1.1` + clearing `Connection` — enables keepalive connections to the upstream instead of opening/closing a TCP connection per request (major performance factor for high-throughput proxying — the default `proxy_http_version` is actually `1.0`, which does NOT support keepalive, a very common perf mistake).
- Timeouts: `connect` (TCP handshake to upstream), `send` (writing request body to upstream), `read` (time between successive reads of the response) — **not** an overall request deadline; a slow-but-trickling response can exceed intuitive expectations.

## WebSocket Proxying 🔥
```nginx
location /ws/ {
    proxy_pass http://ws_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade    $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_read_timeout 3600s;      # long-lived connections need long/no timeout
}
```
`$http_upgrade` carries the client's `Upgrade` header value through; if it's empty (a normal HTTP request to the same path), you'd want `Connection ""` instead of `"upgrade"` — commonly handled with a `map` (see Part 9).

## Common Mistakes
- Not forwarding `X-Forwarded-Proto` → backend generates insecure redirect loops behind HTTPS-terminating Nginx.
- Leaving `proxy_http_version` at default 1.0 → no upstream keepalive → wasted TCP handshakes → higher latency and backend connection exhaustion under load.
- Not raising `client_max_body_size` for upload endpoints → 413 errors.
- Forgetting `proxy_read_timeout` for slow/streaming endpoints → premature 504s.

## Troubleshooting
- **502 Bad Gateway** → Nginx couldn't get a valid response from upstream (upstream down, connection refused, malformed response). Check `error_log`, check if backend is actually listening (`ss -tlnp`), check `proxy_pass` target correctness.
- **504 Gateway Timeout** → upstream accepted the connection but didn't respond within `proxy_read_timeout`. Check backend logs/latency, DB query times, or raise the timeout for known-slow endpoints (with caution).
- **499** → client closed the connection before Nginx finished — usually a client-side timeout or user navigating away; check average backend response times.

## Lab
Stand up a simple Node.js/Django/FastAPI app on port 8000; reverse proxy it through Nginx with full header forwarding; verify with the app's own request-logging that `X-Forwarded-For`/`X-Real-IP`/`Host` arrive correctly; then intentionally stop the backend and observe the resulting 502 in `error_log`.

## Interview Qs
- Why must you explicitly set `proxy_set_header Host $host`? What's the default behavior without it?
- Explain the difference between `proxy_connect_timeout`, `proxy_send_timeout`, and `proxy_read_timeout`.
- Why does `proxy_http_version 1.1` matter for performance?
- Walk through diagnosing a 502 vs a 504 — what's the difference in what happened?

---

# PART 7 — LOAD BALANCING 🏭⭐

## Concepts & Algorithms

```nginx
upstream app_backend {
    least_conn;                       # algorithm selection (default: round robin if omitted)

    server 10.0.1.10:8080 weight=3;
    server 10.0.1.11:8080 weight=1;
    server 10.0.1.12:8080 backup;      # only used when all non-backup servers are down
    server 10.0.1.13:8080 down;        # manually marked out of rotation

    keepalive 32;                      # keepalive connection pool to upstreams
}
```

| Algorithm | Directive | Behavior | Best for |
|---|---|---|---|
| Round Robin | (default, no directive) | Requests distributed sequentially | Uniform, stateless backends |
| Weighted RR | `weight=N` on server | More requests to higher-weight servers | Heterogeneous server capacity |
| Least Connections | `least_conn;` | Sends to server with fewest active connections | Uneven request duration |
| IP Hash | `ip_hash;` | Same client IP → same server (session persistence) | Sticky sessions without shared session store |
| Generic Hash | `hash $request_uri consistent;` | Custom key-based distribution, `consistent` minimizes remapping on server add/remove | Cache-friendly routing, sharding |

### Health & Failure Handling
`max_fails=3 fail_timeout=30s;` on a `server` line — after 3 failed attempts within 30s, Nginx marks that server temporarily unavailable and stops sending it traffic for 30s, then retries. **Open-source Nginx has no active health checks** (no periodic proactive probing) — it only reacts passively to failed requests. Active health checks require **Nginx Plus** or external tooling (e.g., a sidecar, or Kubernetes readiness probes removing a pod from the Service before Nginx ever sees it).

```
                         ┌───────────────┐
Client ──HTTPS──▶  Nginx │  reverse proxy │──▶ upstream { server A, B, C }
                         │  + TLS term    │        │      │      │
                         │  + LB          │        ▼      ▼      ▼
                         └───────────────┘      App A   App B   App C
```

## Common Mistakes
- Using `ip_hash` behind a NAT/corporate proxy where thousands of users share one IP → uneven load, one server gets hammered.
- Not setting `keepalive` on the upstream block → no connection reuse to backends → excess TCP handshake overhead at scale.
- Confusing `weight` semantics — weight is *relative*, not a percentage; `weight=3` vs `weight=1` gives roughly 3:1 not "30%".

## Troubleshooting
- One backend consistently overloaded → check `least_conn` vs round robin choice, check weights, check for a "hot" cache key with `hash`.
- All traffic to one server despite multiple defined → check `down`/`backup` flags weren't accidentally left on other servers, check DNS resolution if using hostnames (Nginx resolves upstream hostnames at *start/reload* time by default unless using `resolver` + dynamic resolve — stale DNS is a classic subtle bug in containerized/K8s environments).

## Lab
Run 3 identical backend containers; configure round robin, then switch to `least_conn`, then `ip_hash`; use `ab` or `hey` to generate load and observe distribution via each backend's own request counter/log.

## Interview Qs
- Compare round robin, least_conn, and ip_hash — when would you choose each?
- How does open-source Nginx detect a failed upstream, and what's the limitation vs Nginx Plus active health checks?
- What does `weight` actually control, precisely?
- Why might upstream hostname resolution be stale in a Kubernetes environment, and how do you fix it?

---

# PART 8 — SSL/TLS & HTTPS 🏭⭐

## TLS Handshake, Conceptually
Client and server negotiate a TLS version and cipher, server presents its certificate (proving identity via a CA-signed chain), a shared symmetric session key is established (via the negotiated key exchange — modern TLS 1.3 does this in effectively 1 round trip vs TLS 1.2's 2), and encrypted application data (HTTP) flows afterward.

## Production SSL Configuration
```nginx
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers off;      # TLS 1.3 negotiates this client-side; irrelevant/counterproductive to force server order on modern stacks

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;             # ticket key rotation is an ops burden; disable unless you manage it properly

    ssl_stapling on;                     # OCSP stapling: server fetches & caches the revocation status
    ssl_stapling_verify on;
    resolver 1.1.1.1 8.8.8.8 valid=300s;

    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
}

server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;   # force HTTPS
}
```

**Line-by-line:** `ssl_protocols` excludes TLS 1.0/1.1 (deprecated, insecure, and PCI-DSS non-compliant); `ssl_session_cache`/`ssl_session_timeout` let repeat clients skip a full handshake (**session resumption**, a real performance win); OCSP stapling lets Nginx proactively attach certificate-revocation-check results to the handshake so clients don't need a separate round trip to the CA; HSTS tells browsers to *never* attempt plain HTTP to this domain again for the `max-age` duration, closing the window for downgrade attacks — but is **hard to undo** (browsers cache it), so roll it out carefully, starting with a short `max-age` before committing to `preload`.

## Let's Encrypt / Certbot Renewal
```bash
certbot certonly --nginx -d example.com -d www.example.com
certbot renew --dry-run              # test the renewal path
# real cron/systemd timer runs:
certbot renew --deploy-hook "systemctl reload nginx"
```
Certificates are valid 90 days; renewal should run well before expiry (certbot's default timer checks twice daily, renews when <30 days remain). **The single most common Nginx-adjacent production incident is a missed cert renewal** (Part 24, Incident #5).

## SSL Termination vs Passthrough
- **Termination (common):** Nginx decrypts TLS, talks plaintext (or re-encrypted) to the backend. Nginx needs the private key.
- **Passthrough (via `stream` module, `ssl_preread`):** Nginx routes encrypted TLS bytes to a backend based on SNI *without* decrypting — used when the backend itself must terminate TLS (e.g., mTLS to the app, or multiple non-Nginx TLS-terminating services behind one IP).

## Common Mistakes
- Forgetting the intermediate certificate chain (`fullchain.pem` not `cert.pem`) → browsers that don't have the intermediate cached show trust errors even though the leaf cert is valid.
- Setting HSTS `preload` before verifying HTTPS works everywhere (subdomains included) → can lock users out of HTTP access for up to a year with no easy rollback.
- Not automating renewal → expiry incidents.

## Troubleshooting
`openssl s_client -connect example.com:443 -servername example.com` to inspect the actual chain served; `curl -vI https://example.com` to see the handshake and headers; `openssl x509 -in cert.pem -noout -dates` to check expiry.

## Lab
Issue a real Let's Encrypt cert via Certbot for a test domain; configure the full production SSL block above; run an SSL Labs test (or `testssl.sh`) against it; verify HSTS and OCSP stapling are actually active with `openssl s_client -status`.

## Interview Qs
- Explain TLS session resumption and why it matters for performance at scale.
- What's OCSP stapling and what problem does it solve?
- Why is `fullchain.pem` required instead of just the leaf certificate?
- What are the risks of enabling HSTS preload, and how do you roll it out safely?
- Difference between SSL termination and SSL passthrough — when would you use passthrough?

---

# PART 9 — WEBSOCKETS & REAL-TIME 🔥

Building on Part 6's WebSocket basics: the key mechanism is the `map` directive handling the `Connection` header correctly for both upgrade and non-upgrade requests on the same location.

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    location /socket.io/ {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```
For **Server-Sent Events (SSE)** and long-polling, the concern shifts to **buffering**: `proxy_buffering off;` is often needed so events stream to the client immediately instead of Nginx waiting to fill a buffer before flushing.

## Common Mistakes
- Default `proxy_read_timeout` (60s) silently killing idle WebSocket/SSE connections — clients see random disconnects every ~60s.
- Leaving `proxy_buffering on` for SSE → chunked events arrive in bursts instead of in real time.

## Troubleshooting
Symptom: WebSocket connects then drops after exactly ~60s → check `proxy_read_timeout`. Symptom: SSE updates delayed/batched → check `proxy_buffering`.

## Interview Qs
- Why is a `map` needed to handle the `Connection` header for WebSocket proxying instead of hardcoding `"upgrade"`?
- Why would you disable proxy buffering for Server-Sent Events?

---

# PART 10 — CACHING 🔥🏭

## Proxy Cache
```nginx
proxy_cache_path /var/cache/nginx/proxy levels=1:2 keys_zone=api_cache:10m max_size=1g inactive=60m use_temp_path=off;

server {
    location /api/products/ {
        proxy_pass http://backend;
        proxy_cache api_cache;
        proxy_cache_key "$scheme$request_method$host$request_uri";
        proxy_cache_valid 200 302 10m;
        proxy_cache_valid 404 1m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_lock on;                 # prevent thundering herd on cache miss
        add_header X-Cache-Status $upstream_cache_status;   # HIT/MISS/EXPIRED/BYPASS/STALE
    }
}
```
- `keys_zone=api_cache:10m` — shared memory zone holding cache *keys/metadata* (not the content itself, which is on disk) — roughly 10MB ≈ 80,000 keys.
- `proxy_cache_use_stale updating` — serve a stale (already-cached) response while a background request refreshes it, instead of making every concurrent client wait — huge for traffic spikes.
- `proxy_cache_lock on` — only one request is allowed to actually go fetch from origin on a cache miss for the same key; others wait for that result instead of stampeding the backend (**thundering herd** protection).

## Cache Purge (open-source limitation)
Open-source Nginx has no built-in "purge by URL" API — you either set short `proxy_cache_valid` TTLs, use a workaround (`proxy_cache_bypass` triggered by a special header, or deleting cache files by key hash on disk), or use Nginx Plus's `proxy_cache_purge` directive.

## When to Cache (and when not to)
- ✅ Public, non-personalized API responses (product catalogs, public content)
- ✅ Static assets (covered in Part 3)
- ❌ Personalized/authenticated responses (unless keyed carefully per-user, which usually isn't worth it — leave that to app-level caching/CDN with proper `Vary` handling)
- ❌ Anything requiring immediate consistency (financial transactions, inventory counts at checkout)

## Common Mistakes
- Caching responses that include `Set-Cookie` or per-user data without excluding them → serving one user's private data to another (serious security bug).
- Cache key not including query string when it should (or including it when it shouldn't) → cache poisoning or 0% hit rate.

## Troubleshooting
Check `X-Cache-Status` header (as configured above) to see HIT/MISS/BYPASS in real time; `du -sh /var/cache/nginx` to watch cache growth; `proxy_cache_valid` mistuned shows up as stale content complaints or low hit ratio in your metrics.

## Interview Qs
- What problem does `proxy_cache_lock` solve?
- How would you safely cache an API that occasionally returns user-specific data?
- Nginx cache vs CDN — where does each belong in the architecture?

---

# PART 11 — PERFORMANCE OPTIMIZATION 🔥🏭

## Identify Bottlenecks Before Tuning (the golden rule)
Never tune blind. Sequence: (1) check `top`/`htop` for CPU (TLS handshakes and gzip are the usual CPU hogs), (2) check `ss -s` for connection state counts, (3) check `error_log` for `worker_connections exhausted` or upstream timeouts, (4) check backend latency directly (is Nginx even the bottleneck, or is it the app/DB?), (5) *then* tune the specific dial that addresses what you found.

## Core Tuning Directives

| Directive | Default | What it does | When to change |
|---|---|---|---|
| `worker_processes` | 1 | Number of worker processes | `auto` (= CPU core count) almost always |
| `worker_connections` | 512 | Max simultaneous connections per worker | Raise (e.g., 4096–65536) for high-concurrency edges; must also raise OS file descriptor limits |
| `worker_rlimit_nofile` | (OS default) | Max open file descriptors per worker | Must exceed `worker_connections × 2` (client + upstream sockets) |
| `keepalive_timeout` | 75s | How long to hold idle client connections open | Lower (e.g., 15-30s) under very high connection churn to free resources faster; higher favors reuse |
| `keepalive_requests` | 1000 | Max requests per keepalive connection before closing | Rarely needs changing |
| `sendfile` | off | Kernel-level zero-copy file transfer | `on` — almost always, for static file serving |
| `tcp_nopush` | off | Batch response headers+body into fewer packets | `on` when `sendfile on` |
| `tcp_nodelay` | on | Disable Nagle's algorithm on keepalive connections | Usually leave `on` (low latency) |
| `gzip_comp_level` | 1 | Compression ratio vs CPU tradeoff | 4-6 is a good balance; 9 rarely worth the CPU |
| `open_file_cache` | off | Cache file metadata (existence, size) to avoid repeat stat() calls | `on` for static-file-heavy sites |

## OS-Level Tuning (paired with Nginx config)
```bash
# /etc/security/limits.conf
nginx soft nofile 65536
nginx hard nofile 65536

# /etc/sysctl.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.ip_local_port_range = 1024 65535
net.ipv4.tcp_tw_reuse = 1
```
Nginx's `worker_connections` alone is meaningless if the OS file descriptor limit or `somaxconn` (listen backlog) caps you first — a very common "why doesn't raising worker_connections help" gotcha.

## Common Mistakes
- Raising `worker_connections` without raising `worker_rlimit_nofile`/OS ulimits → workers silently fail to accept new connections past the OS-imposed ceiling.
- Setting `gzip_comp_level 9` on a CPU-constrained box → trades bandwidth for CPU exhaustion, often a net loss.
- Tuning buffers/timeouts speculatively without profiling → masks the real bottleneck (usually the backend, not Nginx).

## Troubleshooting High Load
- `top`/`htop` → is a worker at 100% CPU (TLS/gzip-bound) or is Nginx idle while backend is slow (I/O-bound elsewhere)?
- `ss -s` → count of connections in each TCP state; a huge `TIME-WAIT` count suggests connection churn (short-lived, non-keepalive connections).
- `nginx -V` to confirm which modules/build flags are compiled in (affects available tuning knobs like `--with-threads`).

## Lab
Load test a static file endpoint with `wrk` or `hey` at increasing concurrency; watch `htop` and `ss -s`; find the point where response times degrade; identify whether it's `worker_connections`, OS file descriptors, or CPU (gzip) that caps you; fix the actual bottleneck; re-test.

## Interview Qs
- Walk through your methodology for diagnosing a "Nginx is slow" ticket — what do you check, in what order?
- Why does raising `worker_connections` sometimes have zero effect?
- Explain the relationship between `sendfile`, `tcp_nopush`, and `tcp_nodelay`.
- How would you tune Nginx to handle 100,000 concurrent connections? (Multi-layered: worker_processes/connections, OS ulimits, sysctl net tuning, possibly multiple Nginx instances behind an LB, keepalive tuning both client-side and upstream-side.)

---

# PART 12 — SECURITY 🏭⭐

## Security Headers
```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
add_header Content-Security-Policy "default-src 'self'; script-src 'self'" always;
```
The `always` flag matters: without it, `add_header` is skipped on error responses (4xx/5xx) by default — meaning your security headers vanish precisely on error pages, which is exactly when some attacks target.

## Hiding Server Identity
```nginx
server_tokens off;    # removes version number from default error pages and Server header
```
Doesn't fully hide that it's Nginx (headers, error page style, timing can still fingerprint it) but removes the easy version-number-based CVE targeting.

## Access Control & Basic Auth
```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.1.0/24;
    deny all;

    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

## Blocking Common Attack Patterns
```nginx
# Block access to hidden files (.env, .git, etc.)
location ~ /\.(?!well-known) {
    deny all;
    return 404;
}

# Block sensitive extensions
location ~* \.(sql|bak|env|log)$ {
    deny all;
    return 404;
}
```

## Request Smuggling / Host Header Attacks
Always define a `default_server` (Part 4) so unexpected/forged Host headers don't hit a real app. Avoid `proxy_set_header Host $http_host` blindly forwarding an attacker-controlled Host without validation when the backend trusts it for things like password-reset link generation — prefer `$host` (which reflects the matched `server_name`, not the raw client-supplied header) when you specifically want to prevent Host header injection into app logic, though note this changes behavior for legitimate multi-domain setups — choose deliberately per app.

## Slowloris / Slow Client Protection
```nginx
client_body_timeout 10s;
client_header_timeout 10s;
send_timeout 10s;
```
These bound how long Nginx will wait on a slow/malicious client trickling data, protecting worker connection slots from being tied up.

## Common Mistakes
- `add_header` without `always` → security headers missing on error pages.
- Leaving `server_tokens on` in production → trivial version fingerprinting.
- No rate limiting on login/auth endpoints (see Part 13) → credential stuffing/brute force exposure.
- Trusting `X-Forwarded-For` from directly-connected clients (not just from a trusted upstream proxy/CDN) → IP spoofing for rate-limit/geo-block bypass; use `set_real_ip_from` + `real_ip_header` only for genuinely trusted proxy hops.

## Troubleshooting
Verify headers with `curl -I`; test hidden file blocking with `curl -I https://site/.env`; test Host header handling with `curl -H "Host: evil.com"`.

## Interview Qs
- Why does the `always` flag on `add_header` matter for security?
- How do you prevent Host header injection attacks in Nginx?
- What's Nginx's built-in defense against Slowloris-style attacks?
- Why is blindly trusting `X-Forwarded-For` dangerous, and how do you configure trust correctly?

---

# PART 13 — RATE LIMITING & TRAFFIC CONTROL 🏭🔥

```nginx
http {
    limit_req_zone  $binary_remote_addr zone=login_limit:10m rate=5r/m;
    limit_req_zone  $binary_remote_addr zone=api_limit:10m   rate=20r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    server {
        location /login {
            limit_req zone=login_limit burst=3 nodelay;
            proxy_pass http://backend;
        }

        location /api/ {
            limit_req zone=api_limit burst=40 nodelay;
            limit_conn conn_limit 10;
            proxy_pass http://backend;
        }
    }
}
```

- **`limit_req_zone`** defines a shared-memory "leaky bucket" keyed by (here) client IP, tracking request rate.
- **`rate=5r/m`** — steady-state allowed rate; requests beyond it are either delayed or rejected depending on `burst`.
- **`burst=3`** — allows short bursts above the steady rate, queued up to 3 extra requests.
- **`nodelay`** — serves burst-allowed requests immediately instead of artificially delaying them to smooth the rate (without `nodelay`, Nginx *queues* excess requests to enforce the average rate, adding latency; with it, it enforces via outright rejection past the burst ceiling instead of queuing delay).
- **`limit_conn`** — caps *simultaneous* connections per key, independent of request rate (protects against connection-exhaustion abuse, e.g., many parallel slow downloads from one IP).

Exceeding limits returns `503` (`limit_req_status`/`limit_conn_status` are configurable, e.g. to 429).

## Common Mistakes
- Applying the same tight rate limit to both login endpoints and general API traffic → legitimate high-traffic users get throttled while attackers just spread across more IPs.
- Keying solely by IP when many real users share one IP (corporate NAT, mobile carrier-grade NAT) → false positives.
- Forgetting `nodelay` when you actually want fast-fail behavior instead of added latency.

## Troubleshooting
Watch `error_log` for `limiting requests, excess...` entries; check `$binary_remote_addr` isn't always resolving the same (e.g., behind a misconfigured proxy where all traffic appears to come from one upstream IP — need `real_ip` config, Part 12).

## Lab
Configure `limit_req` on a test login endpoint at `5r/m` with `burst=2 nodelay`; hammer it with a loop of `curl` requests; observe when `503`s start; check `error_log` for the exact rejection reasoning.

## Interview Qs
- Explain the leaky-bucket model behind `limit_req` and what `burst`/`nodelay` each control.
- How would you protect a login endpoint from brute-force attempts using Nginx alone?
- Difference between `limit_req` and `limit_conn` — what distinct attack does each address?

---

# PART 14 — LOGGING & OBSERVABILITY 🏭

## Structured JSON Logging
```nginx
log_format json_combined escape=json
  '{"time":"$time_iso8601","remote_addr":"$remote_addr",'
  '"request":"$request","status":$status,'
  '"body_bytes_sent":$body_bytes_sent,'
  '"request_time":$request_time,'
  '"upstream_response_time":"$upstream_response_time",'
  '"upstream_status":"$upstream_status",'
  '"http_referer":"$http_referer","http_user_agent":"$http_user_agent",'
  '"http_x_forwarded_for":"$http_x_forwarded_for",'
  '"request_id":"$request_id"}';

access_log /var/log/nginx/access.log json_combined;
```
- `$request_time` — total time Nginx spent on the request (from first byte in to last byte out).
- `$upstream_response_time` — time specifically spent waiting on the backend — comparing these two tells you whether latency is Nginx-side (buffering, TLS, gzip) or backend-side.
- `$request_id` (needs `ngx_http_uuid_module` or a `map`/third-party module) — enables **request tracing/correlation** across Nginx → app → downstream logs.

## Observability Stack Integration
- **Prometheus:** the `nginx-prometheus-exporter` (or `stub_status` + exporter) exposes connection counts, request rates. For richer metrics, `nginx-module-vts` (open-source) or Nginx Plus's native API.
- **Grafana:** dashboard on top of Prometheus metrics — connections active/reading/writing, requests/sec, upstream latency percentiles.
- **Loki/ELK:** ship JSON access logs via Promtail/Filebeat/Fluentd for full-text/structured log search and alerting.
- **CloudWatch:** in AWS, ship logs via the CloudWatch agent or Fluent Bit sidecar (common in ECS/EKS setups).
- **OpenTelemetry:** the `otel` Nginx module (or a sidecar/OpenTelemetry Collector consuming JSON logs) can inject trace context propagation across the proxy boundary.

```nginx
location = /stub_status {
    stub_status;
    allow 127.0.0.1;
    deny all;
}
```
`stub_status` gives: Active connections, accepts/handled/requests totals, Reading/Writing/Waiting counts — the baseline free metric every Nginx dashboard starts with.

## Useful Alerts
- Upstream 5xx rate > threshold (backend degradation)
- `$upstream_response_time` p99 above SLA
- Active connections approaching `worker_connections × worker_processes`
- Certificate expiry <14 days (external check, not from Nginx logs)
- Request rate anomaly (possible DDoS/bot traffic)

## Common Mistakes
- Logging every static asset request at full volume → log noise and disk/ingestion cost; use `access_log off;` for high-volume, low-value paths (health checks, static assets already covered by CDN metrics).
- Not correlating `$request_time` vs `$upstream_response_time` → misattributing backend slowness to Nginx or vice versa.

## Lab
Enable `stub_status`, scrape it manually with `curl`, then wire up `nginx-prometheus-exporter` + a local Prometheus + Grafana dashboard showing active connections and request rate over a load test.

## Interview Qs
- How do you distinguish Nginx-side latency from backend latency using its own logs?
- What does `stub_status` expose, and what are its limitations vs a full metrics exporter?
- How would you implement request tracing across Nginx and multiple backend services?

---

# PART 15 — TROUBLESHOOTING 🏭⭐

For every status code / issue: **Symptoms → Possible Causes → Commands → Logs → Diagnosis → Fix → Prevention**

## HTTP Status Code Reference

| Code | Meaning | Typical Cause |
|---|---|---|
| 400 | Bad Request | Malformed request line/headers, oversized headers, invalid URL encoding |
| 401 | Unauthorized | `auth_basic` failed, or app-level auth header missing |
| 403 | Forbidden | `deny all`, file permissions, `autoindex` off with no index file |
| 404 | Not Found | Wrong `root`/`alias`, `try_files` fallback missing, path typo |
| 405 | Method Not Allowed | Location restricts methods (`limit_except`), or backend rejects the method |
| 408 | Request Timeout | Client too slow sending request (`client_header_timeout`/`client_body_timeout`) |
| 413 | Payload Too Large | `client_max_body_size` too low for the upload |
| 429 | Too Many Requests | `limit_req` triggered |
| 499 | Client Closed Request (Nginx-specific, not standard HTTP) | Client disconnected before Nginx finished responding — often a client-side timeout |
| 500 | Internal Server Error | Backend app error (Nginx is usually just relaying it) |
| 502 | Bad Gateway | Upstream unreachable/refused connection, malformed upstream response, wrong `proxy_pass` |
| 503 | Service Unavailable | All upstreams down/marked failed, or `limit_req`/`limit_conn` rejection, or explicit maintenance `return 503` |
| 504 | Gateway Timeout | Upstream accepted connection but didn't respond within `proxy_read_timeout` |

### Deep Dive: 502 Bad Gateway
- **Symptoms:** clients see 502; intermittent or total.
- **Possible causes:** backend process crashed/not started, wrong IP/port in `upstream`, backend refusing the connection (firewall/SG), backend returning malformed HTTP (e.g., a raw TCP protocol server misconfigured as HTTP), Unix socket permission issues.
- **Commands:** `ss -tlnp | grep <port>` (is anything listening?), `curl http://backend-ip:port/` directly bypassing Nginx, `docker ps`/`kubectl get pods` if containerized.
- **Logs:** `error_log` will show something like `connect() failed (111: Connection refused) while connecting to upstream`.
- **Diagnosis:** isolate whether Nginx or the backend is at fault by hitting the backend directly.
- **Fix:** restart/fix backend, correct `proxy_pass` target, open firewall/security group.
- **Prevention:** health checks (K8s readiness probes removing bad pods from Service before Nginx routes to them), `max_fails`/`fail_timeout` tuning, alerting on upstream 5xx rate.

### Deep Dive: 504 Gateway Timeout
- **Possible causes:** slow DB query, backend deadlock/thread pool exhaustion, network partition, `proxy_read_timeout` too aggressive for a legitimately slow endpoint.
- **Diagnosis:** compare `$upstream_response_time` in logs against `proxy_read_timeout`; check backend APM/logs for the actual slow operation.
- **Fix:** fix the slow backend operation (usually the real fix); only raise the timeout if the slowness is expected/acceptable for that specific endpoint.

## SSL/Certificate Errors
- `SSL_do_handshake() failed` in error_log → protocol/cipher mismatch, or client using very old TLS.
- `curl: (60) SSL certificate problem` → missing intermediate chain (see Part 8).
- Expired cert → browsers show hard interstitial warnings; check `openssl x509 -enddate -noout -in cert.pem`.

## Config & Permission Errors
- `nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)` → another process (or another Nginx instance) already bound that port; `ss -tlnp | grep :80`.
- `13: Permission denied` in error_log for a static file → SELinux context or Unix file permissions; Nginx worker user needs read+execute on the directory chain, read on the file.
- `nginx: [emerg] "location" directive is not allowed here` → structural config error (wrong context nesting).

## DNS Issues (upstream resolution)
Nginx resolves upstream *hostnames* at config load time by default — if you use a hostname that changes IP later (common in Docker/K8s service discovery, or DNS-based backends like RDS endpoints during failover), Nginx keeps using the stale IP until reload, unless you explicitly configure dynamic resolution:
```nginx
resolver 10.0.0.2 valid=10s;
set $backend "internal-service.default.svc.cluster.local";
proxy_pass http://$backend:8080;
```
Using a variable in `proxy_pass` forces Nginx to re-resolve via the configured `resolver` at the `valid` interval instead of once at startup — a critical pattern in Kubernetes/dynamic environments.

## Lab
Deliberately reproduce and fix: a 404 from a bad `alias`, a 502 from a stopped backend, a 413 from a too-small `client_max_body_size`, a 504 from an artificially slow backend endpoint, and a bind() conflict from two Nginx instances on the same port.

## Interview Qs
- Walk through your full diagnostic process for "the site is returning 502 intermittently."
- Why would Nginx keep routing to a dead backend IP after a DNS change, and how do you fix it?
- What's the difference between a 502 and a 504 in terms of what actually happened at the network/protocol level?

---

# PART 16 — NGINX + DOCKER 🏭

```dockerfile
FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY conf.d/ /etc/nginx/conf.d/
COPY dist/ /usr/share/nginx/html/
HEALTHCHECK --interval=10s --timeout=3s CMD curl -f http://localhost/healthz || exit 1
EXPOSE 80 443
```

```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:1.27-alpine
    ports: ["80:80", "443:443"]
    volumes:
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/ssl:ro
      - static_files:/usr/share/nginx/html:ro
    depends_on:
      - app
    networks: [webnet]

  app:
    build: ./app
    expose: ["8000"]           # not published to host — only reachable via nginx on the internal network
    networks: [webnet]
    volumes:
      - static_files:/app/static

networks:
  webnet:

volumes:
  static_files:
```
Container-to-container: Nginx reaches the app via Docker's embedded DNS using the service name (`proxy_pass http://app:8000;`) — Docker Compose's default bridge network resolves service names automatically. **Only expose the app port internally** (`expose`, not `ports`) so it's unreachable from outside the Docker network — Nginx is the sole public entry point.

## Zero-Downtime Reload in Docker
Send `SIGHUP` to the Nginx container (`docker kill -s HUP <container>` or `docker exec <container> nginx -s reload`) after updating a mounted config — avoids a full container restart/downtime. For config changes requiring a new image, use a rolling update strategy at the orchestrator level (or, in plain Compose, `docker compose up -d --no-deps --build nginx` with a health check gating traffic cutover).

## Common Mistakes
- Baking secrets/certs into the image layer instead of mounting them → leaked in image history, and rotation requires a rebuild.
- Not setting a `HEALTHCHECK` → orchestrator can't detect a wedged Nginx process.
- Exposing the backend app's port directly on the host "just for debugging" and forgetting to remove it → bypasses Nginx's security controls entirely.

## Lab
Build the compose stack above with a real app; verify the app port is NOT reachable from the host directly (`curl localhost:8000` should fail, `curl localhost:80` through Nginx should succeed); practice a zero-downtime config reload via `docker exec ... nginx -s reload`.

## Interview Qs
- How does Nginx resolve a backend service name in Docker Compose, and what's the mechanism behind it?
- Why should the backend app's port not be published to the host?
- How do you achieve a zero-downtime config change for Nginx running in Docker?

---

# PART 17 — NGINX + KUBERNETES 🏭🔥

## Two Distinct Things That Get Confused
1. **Nginx as a plain Deployment/Pod** — you manage config via ConfigMap, just like any containerized app.
2. **Nginx Ingress Controller** — a specialized, cluster-wide component that watches `Ingress` resources and dynamically generates Nginx config for *all* apps in the cluster. Most production K8s clusters use the latter.

### Nginx Ingress Controller — Core Concepts
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "20"
    nginx.ingress.kubernetes.io/affinity: "cookie"        # sticky sessions
spec:
  ingressClassName: nginx
  tls:
    - hosts: ["app.example.com"]
      secretName: app-tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: api-service
                port:
                  number: 80
```
- **IngressClass** — lets a cluster run multiple ingress controllers side by side (e.g., `nginx` and an internal-only one), each watching only Ingress resources referencing it.
- **Annotations** — the escape hatch for anything not in the plain Ingress spec (rewrites, rate limiting, sticky sessions, custom snippets) — this is *the* Nginx-Ingress-specific knowledge that differs from vanilla Nginx config syntax.
- Under the hood, the controller pod runs an actual Nginx process and regenerates/reloads its config whenever watched Ingress/Service/Endpoint objects change — same reload mechanics as Part 2, just automated.

### Sticky Sessions, TLS, Health Checks
- Sticky sessions via `affinity: cookie` annotation (uses a generated cookie, similar concept to `ip_hash` but cookie-based, more NAT-friendly).
- TLS via a `Secret` of type `kubernetes.io/tls`, often issued by `cert-manager` (the K8s-native Certbot equivalent) which auto-renews and rotates the Secret — Nginx Ingress watches the Secret and reloads automatically on change.
- Health: Kubernetes readiness probes remove unhealthy pods from the Service's Endpoints *before* the Ingress controller ever considers them — this is functionally your "active health check," implemented at the K8s layer, not inside Nginx itself.

### Nginx Ingress Controller vs Gateway API
The **Gateway API** is the newer, more expressive, role-oriented successor to Ingress (separates infrastructure/platform-team concerns from app-team routing concerns, supports more protocols and richer matching natively without vendor annotations). Nginx now ships a Gateway API implementation too; **know that Ingress's annotation-heavy model is being superseded**, but Ingress remains extremely common in production today and is worth deep fluency in.

## Common Mistakes
- Forgetting `ingressClassName` in multi-controller clusters → Ingress resource silently ignored by every controller (or picked up by the wrong one).
- Relying on `rewrite-target` regex captures incorrectly → subtle path-mangling bugs that only show up for specific nested routes.
- Not setting resource `limits`/`requests` on the Ingress controller pods → it gets OOM-killed or CPU-throttled under traffic spikes, taking down cluster ingress entirely.

## Troubleshooting
`kubectl describe ingress <name>` for events/annotation parsing errors; `kubectl logs -n ingress-nginx <controller-pod>` for the actual generated Nginx config and reload events; `kubectl exec` into the controller pod and run `nginx -T` to see the live generated config, exactly like debugging plain Nginx.

## Interview Qs
- What's the difference between running Nginx as a plain Deployment vs using the Nginx Ingress Controller?
- How does the Ingress Controller achieve "active health checking" despite open-source Nginx lacking that feature natively?
- Explain how TLS certificate rotation works with cert-manager + Nginx Ingress.
- Nginx Ingress vs Gateway API — what's changing and why?

---

# PART 18 — NGINX + AWS 🏭

## Common Architectures

```
Architecture A (simple):
Internet → Route 53 → EC2 (Nginx, public IP, SG allows 80/443) → App (localhost or same box)

Architecture B (HA, TLS at ALB):
Internet → Route 53 → ALB (ACM cert, TLS termination) → Target Group → EC2s (Nginx, HTTP only, private subnet)
                                                                             │
                                                                             └─▶ App

Architecture C (CDN + edge):
Internet → CloudFront (caches static, WAF) → S3 (static assets)
                    │
                    └──(dynamic paths)──▶ ALB → Nginx (private) → App

Architecture D (multi-AZ HA):
Route 53 (health-checked) → ALB (spans 2+ AZs) → Auto Scaling Group of Nginx/app EC2s across AZs
```

## Nginx behind ALB — Key Considerations
- ALB typically terminates TLS (via **ACM**, free managed certs, auto-renewed — often simpler than running Certbot yourself) and forwards plain HTTP to Nginx targets — Nginx then only needs `listen 80;` and must trust `X-Forwarded-Proto`/`X-Forwarded-For` set by the ALB rather than the raw client connection.
- Health checks: ALB Target Group health check hits a specific path (e.g., `/healthz`) on Nginx — configure a lightweight location that doesn't proxy to a potentially-slow backend for this, or that does a real (but fast) upstream check if you want the health check to reflect actual app health.
- Security Groups: ALB's SG allows 443 from the internet; the EC2/Nginx SG allows 80 **only from the ALB's SG** (not `0.0.0.0/0`) — Nginx is not directly internet-reachable in this pattern.

## When to Use ALB vs Nginx vs CloudFront

| Need | Best fit |
|---|---|
| Simple L7 routing, AWS-native health checks, auto-scaling integration, managed TLS | **ALB** |
| Custom routing logic, advanced rate limiting, WebSocket edge cases, fine-grained header manipulation, non-AWS portability | **Nginx** |
| Static asset caching at global edge locations, DDoS absorption at the edge, WAF integration | **CloudFront** |
| Real production stacks | Usually **all three together**: CloudFront in front for caching/WAF, ALB for AZ-spanning L7 LB + managed TLS + health checks, Nginx behind ALB for app-specific routing/rate-limiting/proxy logic ALB can't express |

## Common Mistakes
- Opening Nginx's SG to `0.0.0.0/0` when it should only accept traffic from the ALB — removes a real defense-in-depth layer.
- Terminating TLS at *both* ALB and Nginx redundantly without a clear reason (extra cert management overhead, usually unnecessary unless you need end-to-end encryption for compliance).
- ALB health check path proxies through to a slow/DB-dependent backend endpoint → false-positive unhealthy marks during normal backend load, causing unnecessary target flapping.

## Interview Qs
- When would you choose ALB over running your own Nginx for load balancing, and vice versa?
- How do TLS termination responsibilities typically split between CloudFront, ALB, and Nginx in a layered AWS architecture?
- Why should Nginx's security group only allow traffic from the ALB's security group?

---

# PART 19 — NGINX HIGH AVAILABILITY 🏭🔥

## Active-Passive with Keepalived (self-managed, non-cloud pattern)
```
        Virtual IP (VIP): 10.0.0.100
        ┌───────────────┬───────────────┐
        │  Nginx #1     │   Nginx #2    │
        │  (MASTER,     │   (BACKUP,    │
        │   holds VIP)  │   standby)    │
        └───────────────┴───────────────┘
       Keepalived (VRRP) monitors health; on Nginx #1 failure,
       VIP moves to Nginx #2 within seconds.
```
`keepalived.conf` defines a VRRP instance with a priority per node; the higher-priority healthy node holds the floating Virtual IP; a health-check script (e.g., checking `nginx -t`/process liveness/a local curl) determines eligibility.

## Active-Active (cloud-native, more common today)
Multiple Nginx instances behind a cloud LB (ALB/NLB) or DNS-based (Route 53 weighted/latency/failover routing), all actively serving traffic, health-checked independently, Auto Scaling Group replacing unhealthy instances automatically. This is generally **preferred in cloud environments** over Keepalived's VIP-based failover, since cloud LBs already solve this problem more simply and integrate with auto-scaling.

## Zero-Downtime Deployment Patterns
- **Rolling:** replace instances/pods one at a time behind the LB, waiting for each to pass health checks before proceeding.
- **Blue-Green:** stand up a fully separate "green" environment, switch the LB/DNS over atomically, keep "blue" as instant rollback.
- **Canary:** route a small percentage of traffic (via `split_clients`, weighted upstream, or LB-level weighted target groups) to the new version, monitor error rates, ramp up gradually.

```nginx
# Simple canary via weighted upstream
upstream app {
    server app-v1:8080 weight=95;
    server app-v2:8080 weight=5;
}
```

## Common Mistakes
- Relying on Keepalived's VIP failover in a cloud VPC without understanding that gratuitous ARP (how VIP failover normally works) doesn't work the same way in cloud SDN — cloud providers typically require an API call to move an ENI/route instead, needing cloud-specific tooling, not naive on-prem Keepalived.
- No automated health-check-driven removal from rotation → a degraded (not fully down) Nginx/backend keeps receiving traffic and serving errors.

## Interview Qs
- Why is Keepalived's classic VIP failover pattern often unsuitable in cloud VPCs without modification?
- Compare rolling, blue-green, and canary deployment for a zero-downtime Nginx-fronted release.
- Design a highly available Nginx architecture spanning two AWS Availability Zones — what components and why?

---

# PART 20 — NGINX ADVANCED FEATURES 🚀

## `map`, `geo`, `split_clients`
```nginx
# map: conditional variable assignment based on another variable
map $http_user_agent $is_bot {
    default 0;
    ~*bot|crawler|spider 1;
}

# geo: variable based on client IP ranges (no external DB needed for simple cases)
geo $is_internal {
    default 0;
    10.0.0.0/8 1;
    192.168.0.0/16 1;
}

# split_clients: deterministic percentage-based bucketing (for A/B tests, canary)
split_clients "${remote_addr}${http_user_agent}" $variant {
    10% "new";
    *   "old";
}
```
These three are the backbone of **conditional routing without `if`** — Nginx's `if` directive is famously limited and buggy inside `location` blocks ("if is evil" is a well-known community phrase); `map`/`geo`/`split_clients` are the idiomatic, safe alternative for most conditional logic.

## gRPC, HTTP/2, HTTP/3-QUIC
```nginx
server {
    listen 443 ssl http2;
    listen 443 quic reuseport;         # HTTP/3
    ssl_certificate ...;
    add_header Alt-Svc 'h3=":443"; ma=86400';

    location / {
        grpc_pass grpc://backend:50051;
        error_page 502 = /error_grpc;
    }
}
```
gRPC uses HTTP/2 framing natively; `grpc_pass` (not `proxy_pass`) is the correct directive for proxying gRPC traffic, handling trailers and streaming correctly. HTTP/3/QUIC (UDP-based transport) requires a build with QUIC support (mainline Nginx has had experimental/stable support land in recent versions) and is increasingly standard at the CDN/edge layer for reduced connection-establishment latency.

## Stream Module (Layer 4: TCP/UDP proxying)
```nginx
stream {
    upstream postgres_backend {
        server 10.0.1.20:5432;
        server 10.0.1.21:5432;
    }
    server {
        listen 5432;
        proxy_pass postgres_backend;
        proxy_timeout 3600s;
    }

    # TLS passthrough via SNI without decrypting
    map $ssl_preread_server_name $upstream {
        app1.example.com backend1;
        app2.example.com backend2;
    }
    server {
        listen 443;
        proxy_pass $upstream;
        ssl_preread on;
    }
}
```
This is how Nginx load-balances non-HTTP protocols (databases, custom TCP services, raw gRPC without HTTP semantics needed, mail) — a genuinely different context (`stream {}`, sibling to `http {}`, not nested inside it).

## Open-Source Nginx vs Nginx Plus (know this distinction cold)

| Feature | OSS Nginx | Nginx Plus |
|---|---|---|
| Active health checks | ❌ (passive only via `max_fails`) | ✅ |
| Dynamic upstream reconfiguration (no reload) | ❌ | ✅ (`api` module, on-the-fly upstream changes) |
| Cache purging API | ❌ (workarounds only) | ✅ `proxy_cache_purge` |
| Session persistence (sticky cookie, not just ip_hash) | ❌ | ✅ |
| Live activity monitoring dashboard | Basic (`stub_status`) | ✅ Rich built-in dashboard/API |
| JWT validation, OAuth | ❌ (needs Lua/OpenResty or app-level) | ✅ Native |
| Support contract/SLA | ❌ Community only | ✅ |

## Lua / OpenResty
OpenResty embeds LuaJIT into Nginx's request-processing phases, enabling arbitrary scripting for things OSS Nginx can't natively do: custom auth logic, dynamic routing decisions from external data stores, request/response body transformation, custom rate-limiting algorithms. This is the open-source path to many features that are otherwise Nginx-Plus-only or app-gateway-only (e.g., Kong is built on OpenResty).

## Interview Qs
- Why is using `if` inside a `location` block often discouraged, and what should you use instead?
- What's the difference between `proxy_pass` and `grpc_pass`, and why does it matter?
- Explain SNI-based TLS passthrough using the stream module.
- List three concrete features Nginx Plus has that open-source Nginx lacks, and how you'd work around each without Plus.

---

# PART 21 — CI/CD & NGINX 🏭

## Config Validation in CI
```yaml
# example GitHub Actions step
- name: Validate Nginx config
  run: |
    docker run --rm -v $PWD/conf.d:/etc/nginx/conf.d:ro nginx:alpine nginx -t
```
Running `nginx -t` inside a matching container image in CI, *before* merge, catches syntax errors before they ever reach a server — never let a broken config reach `main`/production undetected.

## GitOps Pattern
Config lives in Git → CI validates (`nginx -t`) → CD pipeline (or ArgoCD/Flux for K8s ConfigMaps) applies the change → a reload (not restart) is triggered → post-deploy smoke test (`curl` a known endpoint) confirms success → automatic rollback on failure (revert commit, re-sync).

## Terraform + Nginx
Terraform typically manages the *infrastructure* Nginx runs on (EC2 instances, security groups, ALB target groups, Route 53 records, ACM certs) rather than the Nginx config file contents themselves — though `file` provisioners or user-data templates can template `nginx.conf` from Terraform variables for simple cases; more commonly, config management (Ansible) or a container image build handles the actual config content, with Terraform provisioning the compute/network layer around it.

## Ansible + Nginx
Ansible's `template` module renders `nginx.conf.j2` with environment-specific variables, `copy`s it to target hosts, and a `handler` triggers `nginx -s reload` only when the config actually changed (idempotent, safe to re-run).

## Blue-Green / Canary in CI/CD (tying back to Part 19)
CI builds a new image/config version → deployed to the "green"/canary target group → automated smoke tests run against it directly → traffic gradually shifted (weighted target groups, or DNS weighted routing, or Nginx `split_clients`) → full cutover once error rates confirmed nominal → old version torn down after a bake period.

## Common Mistakes
- Applying config changes directly on production hosts outside of the pipeline ("just SSH in and fix it") → configuration drift, no audit trail, no rollback path.
- Reloading without first validating (`nginx -t`) in the exact same environment/image as production (a config valid on your laptop might reference a module not compiled into the production Nginx build).

## Interview Qs
- Describe a CI/CD pipeline for safely deploying Nginx configuration changes to production.
- Why validate config inside the *same container image* used in production, not just locally?
- How would you implement a canary release for a new backend version using Nginx?

---

# PART 22 — PRODUCTION ARCHITECTURE (Worked Examples) 🏭

### Architecture 1 — Internet → Nginx → Node.js
Traffic flow: client → DNS (A/AAAA record) → Nginx (TLS termination, static assets, reverse proxy) → Node.js (Express/Fastify) on `localhost:3000` or a Unix socket. DNS: single A record, or Route 53 if AWS. TLS: Let's Encrypt via Certbot on the Nginx host. Networking: Node.js bound to `127.0.0.1` only, never exposed publicly. Security: rate limiting on API routes, security headers, `client_max_body_size` for uploads. Logging: JSON access log + Node's own app logs, correlated via `$request_id`. Scaling: vertical first, then horizontal with a load balancer once one box isn't enough — Node's single-threaded nature means you likely also run multiple Node processes (PM2 cluster mode) behind the same Nginx. Failure scenarios: Node process crash → 502s until process manager (PM2/systemd) restarts it — configure `max_fails`/`fail_timeout` if running multiple Node instances, or accept brief downtime with a single instance and prioritize fast process supervision instead.

### Architecture 2 — Internet → Nginx → Django → PostgreSQL
Django needs a WSGI server (Gunicorn/uWSGI) — Nginx never talks to Django's dev server in production. Traffic flow: client → Nginx (TLS, static/media files served directly by Nginx, NOT by Django) → Gunicorn (via Unix socket, lower overhead than TCP for same-host) → Django → PostgreSQL. Critical config: `proxy_pass http://unix:/run/gunicorn.sock;`, static files (`STATIC_ROOT`) served by an Nginx `location /static/` pointing directly at the collected static directory — never proxy static asset requests through the Python app. Failure scenarios: Gunicorn worker timeout on a slow DB query → 504 from Nginx; diagnose via comparing `$upstream_response_time` against the actual DB query time in Django's logs/APM.

### Architecture 3 — CloudFront → ALB → Nginx → Application
Covered in depth in Part 18. Key point: three layers of caching/routing/TLS each have a distinct job — don't duplicate responsibilities (e.g., don't re-implement CloudFront's edge caching logic inside Nginx's proxy cache for the same static assets).

### Architecture 4 — Internet → Nginx → Docker → Multiple Services
Nginx routes by path or subdomain to multiple containerized services on one host (or `docker-compose` stack), each in its own container, Nginx as the single ingress point — see Part 16 for the concrete compose setup. Failure scenarios: one container's health check failing shouldn't take down routing to healthy siblings — isolate `upstream` blocks per service, not one shared upstream.

### Architecture 5 — Internet → AWS ALB → Nginx → Kubernetes
ALB (or an AWS Load Balancer Controller-provisioned NLB) sits in front of the Nginx Ingress Controller's Service (`type: LoadBalancer`); Ingress Controller pods do L7 routing to the correct in-cluster Service based on Ingress rules; Service routes to healthy Pod endpoints. Two layers of load balancing (ALB→Ingress pods, Ingress pods→app pods) — understand both are happening, don't conflate them when troubleshooting.

### Architecture 6 — Users → Cloudflare → Nginx → API → Database
Cloudflare provides edge caching, DDoS mitigation, and (optionally) TLS termination at their edge with a separate "origin certificate" between Cloudflare and your Nginx (full-strict SSL mode) — critical: configure Nginx to trust Cloudflare's `CF-Connecting-IP` header (or restore real IPs via `set_real_ip_from` with Cloudflare's published IP ranges) so rate limiting/logging reflects real clients, not Cloudflare's edge IPs.

### Architecture 7 — High Availability Nginx Architecture
Combines Parts 18-19: multi-AZ Auto Scaling Group of Nginx instances behind an ALB/NLB, each instance identically configured (via a golden AMI or container image + config management), Route 53 health-checked failover for DR across regions if needed, centralized logging (not per-instance disk logs that vanish on instance termination) via CloudWatch/Loki, and automated certificate renewal that doesn't depend on any single instance's local state (centralized cert storage — e.g., ACM, or a shared secrets manager if self-managing Let's Encrypt across a fleet).

---

# PART 23 — HANDS-ON PROJECTS

For each: **Requirements → Architecture → Prerequisites → Implementation Steps → Config → Testing → Failure Scenarios → Troubleshooting → Security Checklist → Performance Checklist.** Below are complete outlines for all 15; build them in order — each builds on the last.

1. **Static website with Nginx** — serve a plain HTML/CSS site; add gzip, cache headers, custom error pages (`error_page 404 /404.html;`). Test: `curl -I` cache headers. Security: `server_tokens off`, hide dotfiles. Performance: `sendfile on`, gzip tuning.
2. **Multiple domains on one Nginx** — 3 server blocks, distinct `server_name`s, one `default_server` catch-all returning 444. Test with `curl -H "Host: ..."`. Failure: forgetting default_server → host header confusion.
3. **HTTPS with Let's Encrypt** — Certbot, full production SSL block (Part 8), HTTP→HTTPS redirect, HSTS. Test with `testssl.sh`/SSL Labs. Failure scenario: simulate expired cert, watch browser behavior, fix renewal automation.
4. **React (or Vue/Angular) + Nginx** — production build, SPA `try_files`, immutable asset caching, `index.html` `no-store`. Test deep-link refresh. Failure: forgetting SPA fallback → 404 on refresh.
5. **Django + Nginx + Gunicorn** — Unix socket, static/media split, `client_max_body_size` for file uploads, systemd unit for Gunicorn. Failure: Gunicorn socket permissions → 502.
6. **Node.js + Nginx** — reverse proxy, PM2 cluster mode, upstream with multiple Node instances, `least_conn`. Failure: kill one Node instance, verify `max_fails` reroutes traffic.
7. **WebSocket application** — Socket.io or raw WS backend, `map` for Connection header upgrade, long `proxy_read_timeout`. Failure: default 60s timeout, watch connections drop, fix it.
8. **Nginx reverse proxy to multiple microservices** — path-based routing (`/api/users/`, `/api/orders/`) to distinct upstreams. Failure: trailing-slash proxy_pass bug — reproduce and fix.
9. **Nginx load balancer** — 3 backend replicas, compare round robin vs `least_conn` vs `ip_hash` under load (`hey`/`ab`), observe distribution.
10. **Rate-limited API gateway** — `limit_req`/`limit_conn`, distinct zones for `/login` vs `/api/`, custom 429 error page, test with a hammering script.
11. **Nginx + Docker Compose full stack** — Nginx + app + Postgres, internal-only app network, HEALTHCHECK, zero-downtime reload.
12. **Nginx + Kubernetes Ingress** — deploy Nginx Ingress Controller, define an Ingress with TLS via cert-manager, sticky session annotation, test rolling deploys behind it.
13. **Nginx + AWS (ALB + EC2 + ACM)** — Nginx on EC2 in private subnet, ALB with ACM cert in front, Security Groups locked down to ALB-only ingress, Auto Scaling Group.
14. **Monitoring with Prometheus/Grafana** — `stub_status` + exporter + dashboard, alert on 5xx rate and connection saturation, load test and watch the dashboard react.
15. **Production-grade HA Nginx architecture** — capstone: combine SSL, rate limiting, caching, security headers, structured JSON logging shipped to a log aggregator, multi-instance behind a load balancer, automated cert renewal, full CI/CD config validation pipeline, and a documented runbook for the top 5 incidents from Part 24.

---

# PART 24 — REAL PRODUCTION INCIDENTS (Diagnose-First) 🏭

*For each: think through Symptoms → likely causes → diagnostic commands BEFORE reading the solution.*

**1. All APIs suddenly return 502.**
Diagnose: check if backend process(es) crashed or a deploy just happened. `ss -tlnp` on backend port, `error_log` for `connect() failed`. → *Fix:* restart/fix backend; *Prevent:* health checks + alerting on upstream 5xx.

**2. Site returns 504 during a traffic spike.**
Diagnose: compare `$upstream_response_time` trend against traffic graph — backend saturating (DB connection pool exhausted, thread pool maxed). → *Fix:* scale backend horizontally, add connection pooling limits, consider `proxy_cache` for cacheable spike traffic; *Prevent:* load testing before expected spikes, autoscaling policies.

**3. WebSocket connections disconnect every ~60 seconds.**
Diagnose: default `proxy_read_timeout 60s` killing idle upgraded connections. → *Fix:* raise `proxy_read_timeout` for the WS location; *Prevent:* document this in your WS-serving location template.

**4. Uploads above 10MB fail with 413.**
Diagnose: `client_max_body_size` default (1m) too low. → *Fix:* raise it at the appropriate scope for the upload endpoint only (not globally, to avoid unbounded upload abuse elsewhere); *Prevent:* include upload size limits in your endpoint design docs and Nginx config review checklist.

**5. SSL certificate expires, site shows security warnings.**
Diagnose: `openssl x509 -enddate -noout` confirms expiry; check certbot renewal timer/cron status (`systemctl status certbot.timer`, `certbot certificates`). → *Fix:* renew manually (`certbot renew --force-renewal`), reload Nginx; *Prevent:* automated renewal + expiry monitoring alert (14-day and 3-day thresholds), independent of Nginx itself.

**6. Nginx worker CPU reaches 100%.**
Diagnose: `top -H` (per-thread), check if it's TLS handshake volume, gzip compression level, or a regex-heavy config (catastrophic backtracking in a `location ~` regex is a real, if rare, cause). → *Fix:* offload TLS to a CDN/LB, lower `gzip_comp_level`, audit regex complexity; *Prevent:* load testing with realistic TLS/compression-enabled traffic, not just plaintext benchmarks.

**7. Nginx runs out of connections ("worker_connections exceeded").**
Diagnose: `error_log` explicitly logs this; check `ss -s` for connection state distribution. → *Fix:* raise `worker_connections` + OS `ulimit`/`somaxconn` together (Part 11); *Prevent:* capacity planning against expected peak concurrency, not just average.

**8. One upstream server is overloaded while others are idle.**
Diagnose: check LB algorithm — round robin doesn't account for request cost variance; check for `ip_hash` behind NAT concentrating traffic. → *Fix:* switch to `least_conn`, or address NAT-related IP concentration by moving to cookie-based stickiness; *Prevent:* choose LB algorithm based on actual traffic characteristics up front, not defaults.

**9. Backend becomes "unhealthy" but is actually fine.**
Diagnose: health check endpoint itself is slow/DB-dependent and timing out under normal load, or `max_fails`/`fail_timeout` too aggressive. → *Fix:* make the health check endpoint lightweight and independent of downstream dependencies (or have a clear "deep" vs "shallow" health check distinction); *Prevent:* design health checks deliberately, don't default to hitting `/`.

**10. Incorrect Host header routes traffic to the wrong app / triggers app-level bugs.**
Diagnose: `curl -H "Host: attacker.com"` against your IP directly — does it hit a real app? → *Fix:* add explicit `default_server` catch-all; validate `server_name` matches are exhaustive; *Prevent:* security review checklist item for every new Nginx deployment.

**11. DNS points to the wrong server after a migration.**
Diagnose: `dig`/`nslookup` your domain, compare to expected IP; check TTL — old TTL may mean cached stale records at resolvers. → *Fix:* update DNS record, wait out TTL (or lower TTL proactively before planned migrations); *Prevent:* lower TTL to 300s or less in the days leading up to a planned cutover.

**12. SPA routes return 404 on direct navigation/refresh.**
Diagnose: missing `try_files ... /index.html` fallback (Part 3). → *Fix:* add the fallback; *Prevent:* template SPA deployments with the correct location block from day one.

**13. `nginx -s reload` fails silently / doesn't apply changes.**
Diagnose: run `nginx -t` first — a syntax error means the *old* config keeps running (reload is a no-op on failure, master logs an error but doesn't necessarily surface loudly to whoever ran the command manually if not watching stdout/logs). → *Fix:* always check exit code / run `-t` before `-s reload` in any script or pipeline; *Prevent:* CI validation (Part 21) so bad config never even reaches the reload step.

**14. Intermittent random 5xx errors correlate with deploys.**
Diagnose: check if it's a brief window during rolling restarts where old connections are dropped mid-flight (not a truly graceful drain), or a race where Nginx starts routing to a new pod before its app has finished initializing. → *Fix:* proper readiness probes (K8s) or a startup delay/warm-up check before adding an instance to rotation; ensure graceful shutdown handling in the app (finish in-flight requests on SIGTERM) matched with Nginx's/LB's deregistration delay.

**15. Static assets served with wrong MIME type (browser refuses to execute JS/CSS).**
Diagnose: check `mime.types` inclusion (`include mime.types;` missing or file served via a location that doesn't inherit it, e.g., a custom `default_type` override). → *Fix:* verify `include /etc/nginx/mime.types;` is present in `http {}`; *Prevent:* don't hand-roll `Content-Type` overrides without understanding the default mapping first.

**16. Rate limiting blocks legitimate users during a traffic surge (e.g., product launch).**
Diagnose: `limit_req`/`limit_conn` zones sized for "normal" traffic, not launch-day traffic; `error_log` shows widespread "limiting requests" entries for real user IPs. → *Fix:* temporarily raise limits with a pre-planned change, or use a `map`-based allowlist for known-good traffic patterns; *Prevent:* capacity-plan rate limits against expected peak events, not just steady-state.

**17. Log disk fills up, Nginx (or the whole host) becomes unstable.**
Diagnose: `df -h`, `du -sh /var/log/nginx`; check `logrotate` config/cron actually running. → *Fix:* clear/rotate logs, fix `logrotate`; *Prevent:* ship logs off-box (Part 14) instead of relying on local disk retention, set up disk-usage alerting.

**18. Config works on staging, breaks in production.**
Diagnose: environment drift — different Nginx version/module set, different OS-level limits, different backend hostnames not updated. → *Fix:* reconcile the diff (`nginx -V` comparison, config diff); *Prevent:* use identical images/config-management across environments, environment-specific values via templating, not hand-edited divergent files.

**19. A single slow backend endpoint degrades the entire site's perceived performance.**
Diagnose: `$upstream_response_time` per-location breakdown shows one path dominating; check if it's sharing a worker/connection pool bottleneck with unrelated fast endpoints. → *Fix:* isolate the slow endpoint's `upstream`/`keepalive` pool if it's genuinely resource-starving others, fix the underlying slow operation; *Prevent:* per-endpoint latency monitoring/alerting, not just aggregate.

**20. Certificate renewal succeeds but Nginx keeps serving the old (soon-to-expire) cert.**
Diagnose: certbot renewed the files on disk, but nothing reloaded Nginx (worker processes cache the certificate in memory at load time — a file update alone does nothing until reload). → *Fix:* `nginx -s reload`; *Prevent:* always pair `certbot renew` with a `--deploy-hook "systemctl reload nginx"` so it's automatic, never a manual afterthought.

---

# PART 25 — COMMANDS CHEAT SHEET 🏭

## Nginx-Specific
| Command | Purpose |
|---|---|
| `nginx -v` / `nginx -V` | Version / version + compiled-in modules & build flags |
| `nginx -t` | Test config syntax |
| `nginx -T` | Test + dump full merged config |
| `nginx -s reload\|reopen\|quit\|stop` | Signal the master process |
| `systemctl status\|start\|stop\|restart\|reload nginx` | Service management (systemd distros) |
| `journalctl -u nginx -f` | Follow systemd journal logs for Nginx |
| `tail -f /var/log/nginx/access.log /var/log/nginx/error.log` | Live log tailing |

## Network / Connection Inspection
| Command | Purpose |
|---|---|
| `ss -tlnp` | Listening TCP sockets + owning process |
| `ss -s` | Summary of connection states (ESTABLISHED, TIME_WAIT, etc.) |
| `netstat -anp \| grep nginx` | (legacy alternative to `ss`) |
| `curl -I https://site` | Fetch headers only — quick header/cache/status check |
| `curl -v https://site` | Full verbose request/response, including TLS handshake |
| `curl -H "Host: x.com" https://ip/` | Test virtual host routing directly against an IP |
| `dig example.com` / `nslookup example.com` | DNS resolution check |
| `dig +short example.com` | Just the resolved IP(s) |

## TLS / Certificates
| Command | Purpose |
|---|---|
| `openssl s_client -connect host:443 -servername host` | Inspect live TLS handshake & cert chain |
| `openssl x509 -in cert.pem -noout -dates` | Cert validity window |
| `openssl x509 -in cert.pem -noout -text` | Full cert details |
| `certbot certificates` | List managed certs & expiry |
| `certbot renew --dry-run` | Test renewal without actually renewing |

## Process / Performance
| Command | Purpose |
|---|---|
| `ps aux \| grep nginx` | List Nginx processes (master + workers) |
| `top` / `htop` | Live CPU/memory usage per process |
| `lsof -p <pid>` | Open file descriptors for a worker (diagnose fd exhaustion) |
| `lsof -i :80` | What's using port 80 |

## Text Processing for Log Analysis
| Command | Purpose |
|---|---|
| `grep "502" access.log \| wc -l` | Count occurrences |
| `awk '{print $1}' access.log \| sort \| uniq -c \| sort -rn \| head` | Top client IPs by request count |
| `awk '{print $9}' access.log \| sort \| uniq -c` | Status code distribution |
| `sed -n '100,150p' access.log` | Print a specific line range |
| `tail -n 200 -f error.log` | Last 200 lines, then follow |
| `less +F access.log` | Follow mode inside `less` (pause with Ctrl+C, resume with `F`) |

---

# PART 26 — INTERVIEW PREPARATION

*(A curated, high-signal selection across levels — the "must be able to answer cold" set, spanning the topics above. Use the interview questions embedded within each Part above as your full 140+ question bank; this section adds scenario-based synthesis questions that combine multiple parts.)*

### Beginner-Level Signal Questions
- What is Nginx and what problem was it originally designed to solve?
- Explain the master/worker process model.
- What's the difference between a reverse proxy and a forward proxy?
- Where does Nginx look for configuration files by default, and how does `sites-enabled` relate to `sites-available`?
- What does `nginx -t` do and why is it important before reloading?

### Intermediate-Level Signal Questions
- Explain `root` vs `alias` with a concrete example of how a request path resolves differently under each.
- Walk through Nginx's location-matching algorithm precisely, including `^~` and regex precedence.
- Why must you explicitly forward `Host`, `X-Real-IP`, `X-Forwarded-For`, and `X-Forwarded-Proto` when reverse proxying?
- What's the difference between `proxy_connect_timeout`, `proxy_send_timeout`, and `proxy_read_timeout`?
- Explain the SPA `try_files` pattern and why it's needed.

### Advanced-Level Signal Questions
- Compare load balancing algorithms (round robin, least_conn, ip_hash, hash) and when you'd choose each.
- Explain TLS session resumption, OCSP stapling, and why each matters for production performance/security.
- How does `limit_req`'s leaky bucket model work, and what do `burst`/`nodelay` control?
- Diagnose: "our API intermittently returns 502 under load" — walk through your full process.
- Explain how Nginx caching's `proxy_cache_lock` and `proxy_cache_use_stale` protect against thundering herd during a cache miss/backend blip.

### Senior DevOps-Level Signal Questions
- Design a zero-downtime deployment strategy for a Nginx-fronted service, including config changes, not just app releases.
- How would you scale Nginx to handle 100,000 concurrent connections? Discuss every layer (Nginx config, OS tuning, architecture).
- Explain how you'd integrate Nginx access logs into a full observability stack (metrics + logs + traces) and what SLO-relevant signals you'd extract.
- What's your incident response process for a certificate expiry causing a production outage — both the immediate fix and the long-term prevention?
- Compare self-managed Nginx vs ALB vs CloudFront for a given workload, and justify a layered architecture decision.

### Senior Platform Engineer-Level Signal Questions
- Design a multi-tenant Nginx Ingress setup in Kubernetes supporting per-team rate limits, TLS via cert-manager, and canary releases — what annotations/resources would you use and why?
- Explain the architectural tradeoffs between open-source Nginx, Nginx Plus, and OpenResty/Lua for building an internal API gateway — when would each be the right call?
- How would you implement blue-green deployment for the Nginx configuration itself (not just the backend app) across a fleet of instances?
- Walk through designing a globally distributed, highly available Nginx-fronted architecture spanning multiple AWS regions, including DNS failover, cert management, and cache coherency considerations.
- Nginx Ingress vs Gateway API — what's your migration strategy and what breaks/changes?

*(Full topic-by-topic question banks are embedded at the end of every Part above — Parts 1–21 collectively contain 100+ additional grounded questions with answers/reasoning already worked through in context.)*

---

# PART 27 — 30-DAY LEARNING ROADMAP (2 hrs/day)

## Week 1 — Fundamentals (Parts 1–5)
| Day | Topics | Theory | Hands-on | Practice/Lab | Outcome |
|---|---|---|---|---|---|
| 1 | Architecture, process model, event loop (1.1–1.2) | 45m | 75m | Install Nginx, inspect processes with `ps`/`top`, kill a worker and observe respawn | Explain master/worker model from memory |
| 2 | Directory structure, install, config testing (1.3, 2.3) | 30m | 90m | Lab 1 + Lab 2 (break/fix config, `nginx -t`/`-T`) | Comfortable navigating `/etc/nginx` and reloading safely |
| 3 | Contexts, inheritance, core directives (2.1–2.2) | 60m | 60m | Build a multi-context config; reproduce the `add_header` inheritance gotcha | Understand directive inheritance rules |
| 4 | Static web server (Part 3) | 30m | 90m | Deploy a static site with gzip + cache headers; verify with `curl -I` | Correct caching for hashed vs non-hashed assets |
| 5 | Server blocks & virtual hosting (Part 4) | 30m | 90m | 3 domains on one Nginx, test with `curl -H Host:` | Correctly route by Host header, understand default_server |
| 6 | Location matching deep dive (Part 5) | 60m | 60m | Build the 4-location worked example, predict then verify each match | Explain the exact location-matching algorithm |
| 7 | Review + SPA deployment project | 30m | 90m | Project 1, 2, 4 from Part 23 | Ship a real static/SPA site behind Nginx correctly |

## Week 2 — Reverse Proxy, SSL, Load Balancing (Parts 6–8, 22)
| Day | Topics | Theory | Hands-on | Practice/Lab | Outcome |
|---|---|---|---|---|---|
| 8 | Reverse proxy fundamentals, header forwarding (6.1) | 45m | 75m | Proxy a real Node/Django/FastAPI app, verify headers arrive correctly | Explain why each proxy_set_header matters |
| 9 | Timeouts, buffering, keepalive to upstream (6.1) | 45m | 75m | Simulate a slow backend, tune timeouts, observe 504 threshold | Diagnose timeout-related failures |
| 10 | WebSocket proxying (6.2, Part 9) | 30m | 90m | Project 7 — real WebSocket app behind Nginx | Fix the classic 60s WS disconnect bug |
| 11 | Load balancing algorithms (Part 7) | 45m | 75m | Project 9 — 3 backends, compare RR/least_conn/ip_hash under load | Choose the right algorithm for a given traffic shape |
| 12 | SSL/TLS deep dive, Let's Encrypt (Part 8) | 60m | 60m | Project 3 — real Certbot cert + full production SSL block | Produce an A-rated SSL Labs config |
| 13 | HSTS, OCSP stapling, session resumption | 45m | 75m | Verify each with `openssl s_client`, test HSTS rollout carefully | Explain every SSL directive's purpose |
| 14 | Review + Architecture 1 & 2 (Part 22) | 30m | 90m | Build Django+Gunicorn+Nginx and Node+Nginx end-to-end | Two working production-style stacks |

## Week 3 — Security, Performance, Troubleshooting (Parts 10–15)
| Day | Topics | Theory | Hands-on | Practice/Lab | Outcome |
|---|---|---|---|---|---|
| 15 | Caching (Part 10) | 45m | 75m | Configure proxy_cache, test HIT/MISS via `X-Cache-Status`, simulate thundering herd | Explain cache_lock and use_stale |
| 16 | Security headers, access control, attack mitigation (Part 12) | 45m | 75m | Apply full security header set, block hidden files, test with curl | Harden a real config |
| 17 | Rate limiting (Part 13) | 45m | 75m | Project 10 — rate-limited API gateway, hammer test with a script | Explain leaky bucket, burst, nodelay precisely |
| 18 | Performance tuning methodology (Part 11) | 60m | 60m | Load test with `hey`/`wrk`, identify real bottleneck before tuning | Diagnose-first tuning discipline |
| 19 | Logging & observability (Part 14) | 45m | 75m | Project 14 — Prometheus + Grafana dashboard on a load test | Read your own latency data to find issues |
| 20 | Troubleshooting deep dive (Part 15) | 30m | 90m | Reproduce and fix 502/504/413/404/bind-conflict scenarios cold | Diagnose any status code fast, unaided |
| 21 | Review + incident drills | 30m | 90m | Work through Incidents 1–10 (Part 24) diagnose-first | Confident first-responder instincts |

## Week 4 — Docker, Kubernetes, AWS, Production Architecture (Parts 16–19, 21–22)
| Day | Topics | Theory | Hands-on | Practice/Lab | Outcome |
|---|---|---|---|---|---|
| 22 | Nginx + Docker (Part 16) | 30m | 90m | Project 11 — full compose stack, zero-downtime reload | Container-native Nginx patterns |
| 23 | Nginx + Kubernetes Ingress (Part 17) | 60m | 60m | Project 12 — Ingress Controller + cert-manager + annotations | Debug Ingress with `kubectl describe`/`nginx -T` inside the pod |
| 24 | Nginx + AWS architectures (Part 18) | 45m | 75m | Project 13 — EC2 + ALB + ACM + locked-down Security Groups | Justify ALB vs Nginx vs CloudFront tradeoffs |
| 25 | High availability patterns (Part 19) | 45m | 75m | Design (diagram) an HA architecture; if possible, deploy multi-AZ | Explain active-active vs active-passive tradeoffs |
| 26 | Advanced features: map/geo/split_clients, stream module, gRPC (Part 20) | 60m | 60m | Build a canary split with `split_clients`; TCP-proxy a database with `stream` | Know when to reach beyond basic HTTP config |
| 27 | CI/CD integration (Part 21) | 45m | 75m | Add `nginx -t` validation to a CI pipeline; wire a reload-on-deploy hook | Never ship unvalidated config again |
| 28 | Full architecture synthesis (Part 22) | 30m | 90m | Diagram and justify Architectures 5, 6, 7 for a hypothetical real system | Speak fluently about layered production architectures |
| 29 | Incident drills + interview prep | 30m | 90m | Work through remaining incidents (11–20) + Senior-level interview Qs | Ready for a senior DevOps/Platform interview |
| 30 | Capstone project | 15m | 105m | Project 15 — full production-grade HA Nginx stack, documented runbook | End-to-end operational ownership demonstrated |

---

# NGINX SKILL MATRIX

| Level | You Should Be Able To... |
|---|---|
| **Beginner** | Install Nginx, understand master/worker architecture, serve static files, configure basic server blocks, use `nginx -t`/`reload`, understand `root` vs `alias`. |
| **Intermediate** | Configure reverse proxying with correct header forwarding, understand location matching precedence cold, set up HTTPS with Let's Encrypt, configure basic load balancing, deploy SPAs correctly, read access/error logs fluently. |
| **Advanced** | Tune performance methodically (diagnose before tuning), implement caching with thundering-herd protection, configure rate limiting with correct burst/nodelay semantics, harden security headers and access control, troubleshoot every major status code independently, run Nginx correctly in Docker. |
| **Senior DevOps** | Design and operate Nginx in CI/CD pipelines with validated, GitOps-managed config, implement zero-downtime deployment strategies (rolling/blue-green/canary), integrate full observability (metrics/logs/correlation IDs), run incident response for real production failures unaided, tune OS + Nginx together for high concurrency. |
| **Senior Platform Engineer** | Architect multi-layer production systems (CDN + LB + Nginx + app, correctly dividing responsibilities), operate Nginx Ingress Controller at scale in Kubernetes with TLS automation and per-tenant policies, design HA/multi-AZ/multi-region architectures, make informed OSS-vs-Plus-vs-OpenResty tradeoffs, mentor others through the exact reasoning in this document rather than by rote directive lookup. |
| **Expert** | Read and reason about Nginx's actual request-processing phases and event loop behavior to predict edge-case behavior no documentation directly states, extend Nginx via Lua/OpenResty or third-party modules when the platform's built-in feature set is insufficient, and independently design, deploy, secure, optimize, monitor, troubleshoot, and operate Nginx in production without depending on another engineer — the explicit goal you set for this syllabus. |

---

*End of syllabus. Recommended next step: start Week 1, Day 1 — install Nginx and inspect the process tree before writing a single line of config.*
