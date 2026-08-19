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
│   ├── security.toml
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
├── RELEASE_NOTES_0.4.7_ALPHA_1.md
├── RELEASE_NOTES_0.4.7_ALPHA.md
├── RELEASE_NOTES_0.4.7_MODULAR_1.md
├── RELEASE_NOTES_0.4.7_MODULAR_2.md
├── RELEASE_NOTES_0.4.7_MODULAR.md
├── RELEASE_NOTES_0.4.7_PRE_ALPHA.md
├── RELEASE_NOTES_0.4.7_UNSTABLE.md
├── src
│   ├── certificates.rs
│   ├── config
│   │   ├── defaults.rs
│   │   ├── mod.rs
│   │   ├── proxy.rs
│   │   └── security.rs
│   ├── diagnostics.rs
│   ├── error.rs
│   ├── http3.rs
│   ├── main.rs
│   ├── refngn_parser
│   │   └── mod.rs
│   ├── security.rs
│   ├── server
│   │   ├── common.rs
│   │   ├── lifecycle.rs
│   │   ├── logging.rs
│   │   ├── proxy.rs
│   │   ├── responses.rs
│   │   ├── routing.rs
│   │   └── static_files.rs
│   └── server.rs
├── tests
└── www
    └── index.html


```
