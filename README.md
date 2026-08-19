# Refined Engine (`refngn`) 0.4.7 Unstable

**Status: Alpha / technology preview. Not production-ready.**

** PROJECT NOT YET RELEASED **

Refined Engine is a Linux-first HTTP/HTTPS server and reverse proxy written in Rust. It focuses on explicit configuration, dual-stack networking, TLS through rustls, diagnostics, application-layer security and an experimental custom HTTP request-head validator named `refngn-parser`.

For the complete installation, configuration, CLI, troubleshooting and operational reference, read [`manual.md`](manual.md).

## Highlights

- HTTP/1.1 and HTTP/2 on the normal TCP/TLS listeners.
- TLS ALPN negotiation for `h2` and `http/1.1`.
- Configurable TLS 1.3 / TLS 1.2 acceptance.
- Experimental feature-gated HTTP/3/QUIC transport groundwork.
- Explicit IPv4/IPv6 listeners with `IPV6_V6ONLY` for predictable dual-stack binding.
- Per-site SNI certificates, redirects, static serving and reverse proxy routing.
- Structured IPv4/IPv6 multi-upstreams, safe retries, health checks, streaming responses and WebSocket upgrades.
- Central `security.toml`: request limiting, per-IP connection limits, temporary bans and response-aware brute-force detection.
- Route-specific security profile overrides, for example a stricter `/login` profile.
- `refngn-parser` HTTP/1 field-by-field differential comparison; HTTP/2/3 semantic validation.
- `config test`, `test`, `doctor`/`doktor`, `simulate`, certificate and debug commands.

## Protocol configuration

```toml
[tls]
handshake_timeout_seconds = 10
versions = ["1.3", "1.2"]

[http2]
enabled = true
max_concurrent_streams = 100
max_header_list_size = 32768

[http3]
enabled = false
max_request_body_bytes = 8388608
```

HTTP/2 is part of the normal build. HTTP/3 is deliberately experimental and must be compiled with:

```bash
cargo build --release --features http3
```

See [`KNOWN_LIMITATIONS.md`](KNOWN_LIMITATIONS.md) before testing HTTP/3.

## Security profiles

Main config:

```toml
[security]
enabled = true
config = "/etc/refngn/config/security.toml"
```

Site config:

```toml
[site.security]
enabled = true
profile = "default"

[[site.security.route]]
path_prefix = "/login"
profile = "strict"
```

## Build

```bash
cargo build --release
./target/release/refngn --version
./target/release/refngn config test
./target/release/refngn test
./target/release/refngn doctor
```

## Documentation

- [`manual.md`](manual.md) — operator/configuration manual.
- [`config/sites-active/example.com.toml.example`](config/sites-active/example.com.toml.example) — current site example.
- [`config/security.toml`](config/security.toml) — security policy example.
- [`KNOWN_LIMITATIONS.md`](KNOWN_LIMITATIONS.md) — current Alpha limitations.
- [`CHANGELOG.md`](CHANGELOG.md) — release history.

## License

MIT. See [`LICENSE-MIT`](LICENSE-MIT).


## Unstable dependency cleanup

0.4.7 Unstable keeps the Modular.2 configuration/runtime model while reducing dependency duplication. The default TLS stack is standardized on AWS-LC, Reqwest reuses that provider through its no-provider Rustls feature, the direct `socket2` dependency is aligned to 0.6, and `x509-parser` is moved to the 0.18 line. The modular `src/server/` and `src/config/` layout is retained. The next planned phase is 0.4.7 Beta, focused on stabilization, fuzzing, interoperability and load/security regression testing.
