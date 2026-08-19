# Refined Engine 0.4.7 Unstable

0.4.7 Unstable is a dependency and build cleanup release based on the validated 0.4.7 Modular.2 codebase. It intentionally avoids feature expansion.

## Dependency cleanup

- Updated the direct `socket2` dependency from 0.5 to 0.6 so Refngn can share the same major/minor line already used by Tokio/Hyper.
- Kept AWS-LC as the single Rustls crypto provider in the normal build.
- Switched Reqwest to its Rustls `no-provider` feature so it reuses Refngn's selected provider instead of enabling Ring.
- Configured Tokio-Rustls explicitly for AWS-LC, TLS 1.2 and logging instead of relying on provider defaults.
- Configured experimental Quinn/HTTP3 to use AWS-LC as well, preventing Ring from returning when `--features http3` is enabled.
- Updated `x509-parser` from 0.16 to 0.18.1-compatible API range. Refngn uses the still-supported `parse_x509_certificate` API, and the newer line uses `thiserror` 2 instead of keeping the old `thiserror` 1 line alive. Certificate inspection remains part of the normal CLI.
- Retained Reqwest because it is still used for reverse proxy streaming, health checks, URL parsing and diagnostics. Replacing it in this release would be a larger behavioral refactor rather than dependency cleanup.

## Release build

The existing optimized release profile is retained:

```toml
[profile.release]
lto = true
codegen-units = 1
panic = "abort"
strip = true
```

## Compatibility goal

Configuration syntax and runtime behavior should remain compatible with 0.4.7 Modular.2. HTTP/3 remains experimental and opt-in.

## Required validation

After building, install the newly built binary before testing:

```bash
cargo build --release
sudo systemctl stop refngn
sudo install -Dm755 /etc/refngn/target/release/refngn /usr/bin/refngn
/usr/bin/refngn --version
sudo systemctl start refngn
```

Then run the standard config, doctor, HTTP/1.1, HTTP/2, IPv4, IPv6, TLS 1.2/1.3, proxy, security and failover regression checks.
