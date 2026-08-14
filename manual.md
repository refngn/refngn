# Refined Engine 0.4.6-beta.4 — Manual

## 1. Overview

Refined Engine (`refngn`) is a Linux-first HTTP/HTTPS server and reverse proxy written in Rust. The project provides explicit listeners, per-site routing, SNI TLS, redirects, static files, reverse proxying, diagnostics, request limits, and an experimental request-head parser called `refngn-parser`.

This beta introduces explicit IPv4/IPv6 listener families and bracket-free structured multi-upstreams.

## 2. Important beta notes

- `refngn-parser` is experimental. Hyper remains the stable default.
- `hyper-custom` runs additional custom validation and may reject requests when the parsers disagree.
- `custom` is an experimental mode and should be tested before production deployment.
- Incoming proxy request bodies are still collected within configured limits; upstream response streaming is supported.
- Health checks are currently performed during routing attempts rather than by a persistent background health worker.
- Reload is still a graceful generation recycle rather than a fully in-process socket-preserving configuration swap.

See [`KNOWN_LIMITATIONS.md`](KNOWN_LIMITATIONS.md).

## 3. Installation

Build:

```bash
cd /etc/refngn
cargo build --release
```

Optional formatting requires `rustfmt`:

```bash
rustup component add rustfmt
cargo fmt
```

Install:

```bash
sudo systemctl stop refngn
sudo install -Dm755 target/release/refngn /usr/bin/refngn
sudo install -Dm644 packaging/systemd/refngn.service /etc/systemd/system/refngn.service
sudo install -Dm644 packaging/logrotate.d/refngn /etc/logrotate.d/refngn
sudo install -Dm644 packaging/tmpfiles.d/refngn.conf /usr/lib/tmpfiles.d/refngn.conf
sudo systemd-tmpfiles --create
sudo systemctl daemon-reload
sudo systemctl enable --now refngn
```

Ports 80 and 443 require `CAP_NET_BIND_SERVICE`; the packaged systemd unit provides it.

## 4. File layout

```text
/etc/refngn/config/refngn.toml             main configuration
/etc/refngn/config/sites-active/*.toml     active sites
/etc/refngn/config/certs/                  public certificates/chains
/etc/refngn/config/private/                private keys
/var/log/refngn/                           logs and reports
/var/lib/refngn/debug.toml                 debug state
```

Recommended private-key permissions:

```bash
sudo chown root:refngn /etc/refngn/config/private
sudo chmod 0750 /etc/refngn/config/private
sudo chown root:refngn /etc/refngn/config/private/example.com.key
sudo chmod 0640 /etc/refngn/config/private/example.com.key
```

Check access as the service user:

```bash
sudo -u refngn test -r /etc/refngn/config/private/example.com.key && echo OK
```

## 5. Main configuration

### 5.1 IPv4 and IPv6 listeners

The preferred 0.4.6-beta.4 syntax separates the address families explicitly:

```toml
sites_active = "/etc/refngn/config/sites-active"

[[listen.v4]]
address = "0.0.0.0"
port = 80
tls = false
redirect_to_https = false

[[listen.v6]]
address = "::"
port = 80
tls = false
redirect_to_https = false

[[listen.v4]]
address = "0.0.0.0"
port = 443
tls = true

[[listen.v6]]
address = "::"
port = 443
tls = true
```

`[[listen.v4]]` requires an IPv4 address. `[[listen.v6]]` requires an IPv6 address. IPv6 sockets are opened with `IPV6_V6ONLY`, so IPv4 and IPv6 may bind the same port independently.

Check active sockets with:

```bash
sudo ss -lntp | grep -E ':80|:443'
```

Expected dual-stack result:

```text
0.0.0.0:80
[::]:80
0.0.0.0:443
[::]:443
```

### 5.2 Server defaults

```toml
[server]
max_connections = 2048
graceful_shutdown_seconds = 30
header_timeout_seconds = 10
idle_timeout_seconds = 75
max_header_bytes = 32768
max_headers = 100
max_uri_bytes = 8192
request_body_bytes = 67108864
```

### 5.3 Logging

```toml
[logging]
level = "info"
access_log = "/var/log/refngn/access.log"
error_log = "/var/log/refngn/error.log"
format = "json"
```

### 5.4 TLS defaults

```toml
[tls]
handshake_timeout_seconds = 10
```

Certificates themselves are configured per site.

## 6. Site configuration

Basic site:

```toml
[site]
name = "example.com"
server_names = ["example.com", "www.example.com"]
```

## 7. TLS per site

```toml
[site.tls]
enabled = true
default = true
certificate = "/etc/refngn/config/certs/example.com-fullchain.pem"
private_key = "/etc/refngn/config/private/example.com.key"
```

SNI selects the certificate during the TLS handshake. HTTP `Host` routing happens after TLS negotiation.

Only one enabled site should normally use `default = true`.

## 8. Redirects

### HTTP to HTTPS per site

```toml
[site.redirect]
http_to_https = true
https_port = 443
```

### Canonical-domain redirect

```toml
[site.redirect]
http_to_https = true
https_port = 443
target_domain = "www.example.com"
```

Refngn uses a 308 redirect and preserves path and query string.

## 9. Reverse proxy

### 9.1 One backend

```toml
[[site.proxy]]
path_prefix = "/"
upstream = "http://127.0.0.1:8090"
strip_prefix = false
```

### 9.2 Multiple structured backends

```toml
[[site.proxy]]
path_prefix = "/"
strip_prefix = false
timeout_seconds = 30
connect_timeout_seconds = 5
response_header_timeout_seconds = 30
idle_timeout_seconds = 75
max_request_body_bytes = 67108864
max_response_body_bytes = 268435456
streaming = true
websocket = true

[[site.proxy.multi.upstream]]
protocol = "http"
address = "127.0.0.1"
port = 8090

[[site.proxy.multi.upstream]]
protocol = "http"
address = "::1"
port = 8090
```

Structured IPv6 addresses are written without URL brackets. Refngn constructs a valid internal URL automatically.

Route-level fields such as timeouts and body limits should be placed before the nested `[[site.proxy.multi.upstream]]` blocks.

### 9.3 Legacy multiple-URL form

The following remains supported for compatibility:

```toml
upstreams = [
    "http://127.0.0.1:8090",
    "http://[::1]:8090",
]
```

Do not combine `upstream`, `upstreams`, and `site.proxy.multi.upstream` on the same route.

### 9.4 Proxy headers

```toml
[site.proxy.headers]
host = "$host"
x_real_ip = "$remote_addr"
x_forwarded_for = "$proxy_add_x_forwarded_for"
x_forwarded_host = "$host"
x_forwarded_proto = "$scheme"
```

Useful variables:

- `$host` — normalized incoming host without the client-supplied port.
- `$http_host` — incoming Host value.
- `$upstream_host` — selected upstream host and port.
- `$remote_addr` — direct client address.
- `$scheme` — `http` or `https` for the accepted listener.
- `$proxy_add_x_forwarded_for` — append the remote address to an existing chain.
- `off` — do not send the corresponding header.

Only preserve existing forwarding chains when the preceding proxy is trusted.

## 10. Health checks and retries

```toml
[site.proxy.health_check]
enabled = true
path = "/health"
timeout_seconds = 2

[site.proxy.retry]
attempts = 2
retry_connect_errors = true
```

Automatic retry behavior is deliberately conservative. Safe, bodyless methods such as GET/HEAD/OPTIONS may be retried. Requests that could duplicate state-changing operations are not silently replayed.

## 11. Streaming and WebSockets

```toml
[[site.proxy]]
path_prefix = "/"
upstream = "http://127.0.0.1:8090"
streaming = true
websocket = true
```

Upstream response streaming limits memory use for large responses. WebSocket/HTTP Upgrade traffic uses a bidirectional tunnel for HTTP upstreams.

## 12. Request limits

```toml
[site.limits]
max_header_bytes = 32768
max_headers = 100
max_uri_bytes = 8192
request_body_bytes = 16777216
header_timeout_seconds = 10
body_timeout_seconds = 60
idle_timeout_seconds = 75
```

Site values override global server defaults where applicable.

## 13. Security headers

```toml
[site.headers]
strict_transport_security = "max-age=31536000; includeSubDomains"
x_content_type_options = "nosniff"
x_frame_options = "DENY"
referrer_policy = "strict-origin-when-cross-origin"
content_security_policy = "default-src 'self'"

[site.headers.set]
permissions-policy = "camera=(), microphone=()"
```

HSTS is sent only over HTTPS.

## 14. Request IDs

```toml
[site.request_id]
enabled = true
header = "x-request-id"
trust_incoming = false
```

When incoming IDs are not trusted, Refngn generates a new request ID and forwards it to the backend.

## 15. HTTP parser modes

```toml
[site.parser]
mode = "hyper"
```

Modes:

- `hyper` — stable default; Hyper owns HTTP parsing.
- `hyper-custom` — beta; Hyper remains active and `refngn-parser` performs additional request-head validation.
- `custom` — experimental; custom validation is authoritative for supported request-head checks, while Hyper still provides transport/body/keep-alive support in this beta.

Diagnostics intentionally warn for experimental modes.

## 16. CLI reference

### General

```bash
refngn help
refngn --version
refngn run
```

### Configuration

```bash
refngn config test
refngn config test --domain example.com
```

A successful result includes checks such as TOML syntax, HTTP engine, TLS configuration, backend reachability, and conflicting rules.

### Doctor

```bash
refngn doctor
refngn doktor
refngn doctor --domain example.com
```

Doctor checks configuration, parser mode, TLS, DNS, certificates, security rules, request limits, backends, and proxy routes.

### Simulation

```bash
refngn simulate --host example.com --scheme https --path /api/status
```

Reports may be written to a file where supported by the command options.

### Debug

```bash
refngn debug enable
refngn debug disable
refngn debug connections
refngn debug parser
```

### Certificates

```bash
refngn certificates list
refngn certificates check
refngn certificates expiring --days 30
refngn certificates reload
```

### Reload

```bash
sudo refngn reload
sudo systemctl reload refngn
```

## 17. Testing IPv4 and IPv6

Frontend listeners:

```bash
curl -4 -I http://example.com/
curl -6 -I http://example.com/
curl -4 -kI https://example.com/
curl -6 -kI https://example.com/
```

Backend directly:

```bash
curl -v http://127.0.0.1:8090/
curl -g -v 'http://[::1]:8090/'
```

A structured multi-upstream can only be healthy on IPv6 if the backend application itself listens on IPv6.

## 18. Troubleshooting

### Address already in use on `[::]:80`

0.4.6-beta.4 opens `listen.v6` sockets with `IPV6_V6ONLY`. With the new listener syntax, `0.0.0.0:80` and `[::]:80` are expected to coexist.

Check:

```bash
sudo ss -lntp | grep -E ':80|:443'
```

### TLS private-key permission denied

Check the full path:

```bash
namei -l /etc/refngn/config/private/example.com.key
sudo -u refngn test -r /etc/refngn/config/private/example.com.key && echo OK
```

### Backend reachable over IPv4 but not IPv6

If doctor reports `Connection refused` for `[::1]:8090`, Refngn reached the IPv6 socket but no application is listening there. Configure the backend service to bind IPv6 as well.

### Service is not listening

```bash
sudo systemctl status refngn --no-pager
sudo journalctl -u refngn -n 100 --no-pager
```

### Verify the installed binary

```bash
/usr/bin/refngn --version
readlink -f /usr/bin/refngn
```

## 19. Upgrade notes for 0.4.6-beta.4

The main listener syntax changed from flat blocks:

```toml
[[listen]]
address = "0.0.0.0"
port = 80
```

to explicit families:

```toml
[[listen.v4]]
address = "0.0.0.0"
port = 80

[[listen.v6]]
address = "::"
port = 80
```

Update the main configuration before starting this beta.

For multi-backend proxy routes, prefer:

```toml
[[site.proxy.multi.upstream]]
protocol = "http"
address = "::1"
port = 8090
```

The single URL form remains supported.

## 20. Operational guidance

- Always run `refngn config test` before restarting the service.
- Run `refngn doctor --domain <name>` when introducing a new site or backend.
- Keep private keys readable only by the service group.
- Test both `curl -4` and `curl -6` for dual-stack deployments.
- Treat beta parser modes as experimental until tested against your traffic.
- Do not expose backend application ports publicly unless required.
