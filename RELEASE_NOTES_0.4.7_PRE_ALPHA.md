# Refined Engine 0.4.7 Pre Alpha

This release introduces the first experimental Refngn security engine while keeping the HTTP serving protocol at HTTP/1.1.

## New security surface

- Optional central `/etc/refngn/config/security.toml` policy file.
- Reusable named security profiles.
- Per-site profile selection through `[site.security]`.
- Direct-peer per-IP concurrent connection limiting.
- Fixed-window per-IP/per-site request rate limiting with burst allowance.
- HTTP 429 responses include `Retry-After`.
- Temporary blocking after repeated rate-limit violations.
- Response-status-based brute-force detection with progressive temporary blocking.
- `refngn config test` validates security file/profile references.
- `refngn doctor` reports whether the security engine is enabled for a site.

## Important Pre Alpha behavior

- Counters and bans are in memory and reset on restart/reload.
- Security profiles are site-scoped, not route-scoped.
- Client identity is the direct TCP peer IP. Trusted-proxy/CDN client-IP extraction is not implemented yet.
- This is application-layer protection; it cannot absorb a volumetric DDoS that saturates the network uplink.
- HTTP/2 and HTTP/3 are intentionally deferred to 0.4.7 Alpha.

## Validation after installing

```bash
cargo build --release
./target/release/refngn --version
./target/release/refngn config test
./target/release/refngn test
./target/release/refngn doctor
```
