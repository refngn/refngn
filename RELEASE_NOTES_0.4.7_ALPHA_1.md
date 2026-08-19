# Refined Engine 0.4.7 Alpha.1

This is a focused HTTP/2 routing bugfix release.

## Fixed

- HTTP/2 requests now use the decoded `:authority` value for virtual-host selection.
- Valid HTTP/2 requests no longer fall through to `421 Misdirected Request` merely because an HTTP/1.1 `Host` header is absent.
- HTTP/1.x continues to prefer the `Host` header.
- The selected authority is consistently used by redirects, security selection, proxy variables and request logging.

## Test

```bash
cargo build --release
./target/release/refngn --version
./target/release/refngn config test
curl -v --http2 https://example.com/ -o /dev/null
```

A normal routed site should return its expected status (for example `HTTP/2 200`) instead of `421`.
