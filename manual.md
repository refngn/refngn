# Refined Engine 0.4.7 Unstable — Manual

## 1. Overview

Refined Engine (`refngn`) is a Linux-first HTTP/HTTPS server and reverse proxy written in Rust. The project provides explicit listeners, per-site routing, SNI TLS, redirects, static files, reverse proxying, diagnostics, request limits, and an experimental request-head parser called `refngn-parser`.

This Alpha adds HTTP/2, configurable TLS protocol versions, route-specific security profiles, stronger Hyper/refngn-parser differential validation, and experimental feature-gated HTTP/3/QUIC groundwork while retaining the dual-stack and multi-upstream model.

## 2. Important Alpha notes

- HTTP/2 is enabled by default through Hyper; `refngn-parser` remains experimental.
- `hyper-custom` runs additional custom validation and may reject requests when the parsers disagree.
- `custom` is an experimental mode and should be tested before production deployment.
- Incoming proxy request bodies are still collected within configured limits; upstream response streaming is supported.
- Health checks are currently performed during routing attempts rather than by a persistent background health worker.
- HTTP/3 is feature-gated and does not yet use the normal static/proxy routing path; see Known Limitations.
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
/usr/bin/refngn --version
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

The preferred listener syntax separates the address families explicitly:

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
versions = ["1.3", "1.2"]
```

Supported TLS versions in this Alpha are TLS 1.3 and TLS 1.2. HTTP/3 requires TLS 1.3. Certificates themselves are configured per site.

### 5.5 HTTP/2 and HTTP/3

HTTP/2 is enabled on the normal TCP/TLS listeners through Hyper:

```toml
[http2]
enabled = true
max_concurrent_streams = 100
max_header_list_size = 32768
```

TLS listeners advertise `h2` and `http/1.1` through ALPN when HTTP/2 is enabled.

HTTP/3 is experimental and feature-gated:

```toml
[http3]
enabled = false
max_request_body_bytes = 8388608
```

Build the experimental QUIC/H3 code with:

```bash
cargo build --release --features http3
```

In 0.4.7 Unstable the HTTP/3 transport can accept QUIC/H3 requests and pass their decoded request metadata through `refngn-parser` semantic validation, but the normal static/reverse-proxy routing pipeline is not yet connected to H3 streams. Valid H3 requests currently receive HTTP 501.

### 5.6 Central security configuration

Security is optional and loaded from a separate file:

```toml
[security]
enabled = true
config = "/etc/refngn/config/security.toml"
```

If security is enabled, the file must exist and contain the configured default profile. If a site enables security while global security is disabled, configuration validation fails.

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

## 15. Security engine (Alpha)

Attach a central security profile to a site:

```toml
[site.security]
enabled = true
profile = "default"

[[site.security.route]]
path_prefix = "/login"
profile = "strict"
```

Example `/etc/refngn/config/security.toml`:

```toml
[security]
default_profile = "default"

[security.connections]
max_per_ip = 50

[security.profile.default.rate_limit]
enabled = true
requests = 120
window_seconds = 60
burst = 30

[security.profile.default.blocking]
enabled = true
violation_threshold = 10
window_seconds = 60
ban_seconds = 900

[security.profile.default.bruteforce]
enabled = true
failure_statuses = [401, 403]
attempts = 5
window_seconds = 300
initial_block_seconds = 30
max_block_seconds = 3600
```

The longest matching `[[site.security.route]]` path prefix overrides the site's default security profile. This makes it possible to use a stricter profile for `/login` or `/admin` while keeping normal browsing limits more generous.

The rate limiter is currently a fixed-window per-IP/per-profile limiter. `burst` increases the allowed count within a window. Exceeding the rate limit returns HTTP 429 with `Retry-After`. Repeated rate-limit violations may create a temporary IP ban when blocking is enabled.

Brute-force protection observes response status codes. After the configured number of failures within the window, the direct peer IP is temporarily blocked. Repeated blocks increase the penalty up to `max_block_seconds`. Refngn does not inspect credentials or authentication request bodies.

`security.connections.max_per_ip` limits concurrent TCP connections per direct peer IP. Security counters and bans are in memory and reset when Refngn restarts or reloads into a new process.

**Important:** this security layer is intended for HTTP/application-layer abuse. It cannot prevent volumetric DDoS traffic from saturating the machine or upstream network. When Refngn is placed behind a CDN or another reverse proxy, direct-peer IP limiting will see that proxy's IP in this Alpha; trusted-proxy-aware client-IP handling is planned for a later build.

## 16. HTTP parser modes

```toml
[site.parser]
mode = "hyper"
```

Modes:

- `hyper` — stable default; Hyper owns HTTP/1.1 and HTTP/2 parsing.
- `hyper-custom` — experimental; HTTP/1 request heads are reconstructed and compared field-by-field (method, target, version, headers) against `refngn-parser`. HTTP/2 uses decoded semantic validation.
- `custom` — experimental; Refngn validation is authoritative for supported request-head checks, while Hyper still provides the HTTP/1 + HTTP/2 transport/body/keep-alive layer. HTTP/3 uses the optional H3/QUIC transport and semantic validation.

Diagnostics intentionally warn for experimental modes.

## 17. CLI reference

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
refngn test
```

`refngn config test` is the preferred validation command. `refngn test` is the backward-compatible alias. Run one of these before restart/reload; use `--domain` when you want backend/network checks focused on one configured site. A successful result includes checks such as TOML syntax, HTTP engine, TLS configuration, security profile references, backend reachability, and conflicting rules.

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

## 18. Testing IPv4 and IPv6

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

## 19. Troubleshooting

### Address already in use on `[::]:80`

0.4.7 Unstable opens `listen.v6` sockets with `IPV6_V6ONLY`. With the new listener syntax, `0.0.0.0:80` and `[::]:80` are expected to coexist.

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

## 20. Compatibility and upgrade notes for 0.4.7 Unstable

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

Update the main configuration before starting this Alpha.

For multi-backend proxy routes, prefer:

```toml
[[site.proxy.multi.upstream]]
protocol = "http"
address = "::1"
port = 8090
```

The single URL form remains supported.

Security is opt-in. Existing sites continue to run without `[site.security]`. To enable the Alpha security engine, add `[security]` to the main config, install `security.toml`, and attach a profile to the desired sites.

## 21. Operational guidance

- Always run `refngn config test` before restarting the service.
- Run `refngn doctor --domain <name>` when introducing a new site or backend.
- Keep private keys readable only by the service group.
- Test both `curl -4` and `curl -6` for dual-stack deployments.
- Treat custom parser modes and HTTP/3 as experimental until tested against your traffic.
- Start security profiles with generous limits, observe legitimate traffic, then tighten them.
- Remember that security counters/bans reset across restart/reload in this Alpha.
- Keep upstream/provider DDoS protection for attacks that can saturate your network link.
- Do not expose backend application ports publicly unless required.


## Unstable release note

0.4.7 Unstable is a dependency/build cleanup release on top of the Modular.2 source layout. Existing configuration syntax is intended to remain compatible. The default build uses AWS-LC as the single Rustls crypto provider; Reqwest is configured not to select a second provider; the direct `socket2` line is aligned to 0.6; and `x509-parser` is updated to the 0.18 line. HTTP/3 remains experimental and opt-in.

After every release build, install the newly built binary before checking the version or restarting the service:

```bash
sudo systemctl stop refngn
sudo install -Dm755 /etc/refngn/target/release/refngn /usr/bin/refngn
/usr/bin/refngn --version
sudo systemctl start refngn
```
