
## 1. Layer 3 Project Tree

```bash
/etc/refngn
├── Cargo.lock
├── Cargo.toml
├── CHANGELOG.md
├── config
│   ├── certs
│   │   ├── fullchain.cer
│   │   └── README.md
│   ├── private
│   │   ├── README.md
│   │   └── refngn.com.key
│   ├── refngn-local-tls.toml
│   ├── refngn-tls.example.toml
│   ├── refngn.toml
│   └── sites-active
│       ├── default.toml
│       ├── dualstack.example.toml.example
│       ├── example.com.toml.example
│       ├── proxy.example.toml.example
│       └── refngn.com.toml
├── docs
│   ├── PROJECT_DOCUMENTATION.md
│   └── README.md
├── KNOWN_LIMITATIONS.md
├── LICENSE-MIT
├── manual.md
├── packaging
│   ├── logrotate.d
│   │   └── refngn
│   ├── systemd
│   │   └── refngn.service
│   └── tmpfiles.d
│       └── refngn.conf
├── README.md
├── src
│   ├── certificates.rs
│   ├── config.rs
│   ├── diagnostics.rs
│   ├── error.rs
│   ├── main.rs
│   ├── refngn_parser
│   │   └── mod.rs
│   └── server.rs
├── target
│   ├── CACHEDIR.TAG
│   └── release
│       ├── build
│       ├── deps
│       ├── examples
│       ├── incremental
│       ├── refngn
│       └── refngn.d
├── tests
└── www
    └── index.html

```