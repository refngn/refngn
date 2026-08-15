# Refined Engine (`refngn`) 0.4.7 Pre Alpha

**Status: Pre Alpha / technology preview. Do not treat this build as production-ready.**

NOT YET RELEASED, WORK IN Development!

Refined Engine is a Linux-first HTTP/HTTPS server and reverse proxy written in Rust. It focuses on explicit configuration, predictable routing, TLS through rustls, diagnostics, and an experimental custom HTTP request-head parser named `refngn-parser`.

For the complete installation, configuration, CLI, troubleshooting, and operational reference, read [`manual.md`](manual.md).

## Highlights

- Explicit IPv4 and IPv6 listener groups with independent sockets.
- IPv6 listeners use `IPV6_V6ONLY`, allowing IPv4 and IPv6 to bind the same port safely.
- HTTP and HTTPS listeners in one process.
- Per-site SNI TLS certificates.
- Per-site and listener-level HTTP-to-HTTPS redirects.
- Canonical-domain redirects with path and query preservation.
- Static file serving and reverse proxy routing.
- Single or multi-backend proxy routes, including bracket-free IPv6 endpoint configuration.
- Backend health checks, safe retries, streaming upstream responses, and WebSocket upgrades.
- Per-site HTTP limits, security headers, and request IDs.
- `hyper`, `hyper-custom`, and experimental `custom` parser modes.
- `config test`, legacy `test`, `doctor`/`doktor`, `simulate`, certificate, and debug commands.
- Central `security.toml` with reusable per-site security profiles.
- Per-IP request rate limiting, connection limiting, temporary bans, and response-aware brute-force detection.

## Dual-stack listeners

```toml
[[listen.v4]]
address = "0.0.0.0"
port = 80
tls = false

[[listen.v6]]
address = "::"
port = 80
tls = false

[[listen.v4]]
address = "0.0.0.0"
port = 443
tls = true

[[listen.v6]]
address = "::"
port = 443
tls = true
```

`listen.v6` sockets are explicitly IPv6-only. This prevents `[::]:80` or `[::]:443` from also claiming the IPv4 port on Linux.

## Single upstream

```toml
[[site.proxy]]
path_prefix = "/"
upstream = "http://127.0.0.1:8090"
```

## Multi-upstream with IPv4 and IPv6

```toml
[[site.proxy]]
path_prefix = "/"
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

Do not put brackets around structured IPv6 addresses. Refngn converts `::1` to the internal URL form `[::1]` automatically.


## Security profiles (Pre Alpha)

Enable the central security policy in `refngn.toml`:

```toml
[security]
enabled = true
config = "/etc/refngn/config/security.toml"
```

Attach a reusable profile to a site:

```toml
[site.security]
enabled = true
profile = "default"
```

See [`config/security.toml`](config/security.toml) for the example policy. This layer is intended to reduce HTTP/application-layer abuse. It is not a replacement for provider/CDN/upstream protection against volumetric DDoS.

## Build

```bash
cargo build --release
./target/release/refngn --version
./target/release/refngn config test
```

`cargo fmt` is optional and requires the Rust `rustfmt` component.

## Documentation

- [`manual.md`](manual.md) — complete operator and configuration manual.
- [`docs/PROJECT_DOCUMENTATION.md`](docs/PROJECT_DOCUMENTATION.md) — architecture and project design.
- [`KNOWN_LIMITATIONS.md`](KNOWN_LIMITATIONS.md) — current Pre Alpha limitations.
- [`CHANGELOG.md`](CHANGELOG.md) — release history.

## License

MIT. See [`LICENSE-MIT`](LICENSE-MIT).
