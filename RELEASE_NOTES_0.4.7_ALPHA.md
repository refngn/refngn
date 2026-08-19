# Refined Engine 0.4.7 Alpha.1

0.4.7 Alpha.1 moves the 0.4.7 line from the first security preview into protocol and hardening work.

## Added

- HTTP/2 server support through Hyper/hyper-util auto protocol handling.
- TLS ALPN advertises `h2` and `http/1.1` when HTTP/2 is enabled.
- Global `[http2]` limits for concurrent streams and header-list size.
- Configurable TLS versions: TLS 1.3 and TLS 1.2.
- Experimental feature-gated HTTP/3/QUIC transport (`--features http3`).
- HTTP/3 request-head semantic validation through `refngn-parser` before the Alpha test response.
- Route-specific security profile overrides using `[[site.security.route]]`.

## Improved

- HTTP/1 `hyper-custom`/`custom` now compare method, target, version and headers field-by-field with Hyper.
- The old `ParsedHead` dead-code warning is addressed by real field use rather than suppression.
- Safe retry/failover attempts now try at least all configured upstreams.
- `config test` and `doctor` report HTTP/2, HTTP/3 and TLS-version configuration.
- Example site/main/security configurations and documentation updated.

## HTTP/3 status

HTTP/3 remains intentionally experimental in this Alpha. QUIC/H3 handshake and request validation are present, but the normal Refngn static/proxy routing pipeline is not yet connected to H3 streams. Valid H3 requests currently receive a `501 Not Implemented` test response. Do not advertise this build as complete HTTP/3 support.

## Next

The planned 0.4.7 Unstable release is reserved for dependency/crate reduction, crypto-provider cleanup, duplicate dependency reduction and binary/build-size optimization before the next stabilization phase.
