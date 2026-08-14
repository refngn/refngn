# Known limitations — refngn 0.4.6-beta.4

- `refngn-parser` is beta/experimental. Hyper still provides connection management, keep-alive, body framing, chunking, and upgrades. The custom parser is not yet a fully independent wire-protocol engine.
- Parser selection is per site, so Hyper must parse enough of the request to identify the Host before a site-specific parser mode can be selected.
- Full field-by-field Hyper/custom differential comparison is not finished; `ParsedHead` fields may still trigger a compiler dead-code warning.
- Upstream response streaming is implemented, but incoming proxy request bodies are still bounded and collected before forwarding.
- Health checks currently run during routing attempts instead of using a persistent background health-state worker.
- Failover is deterministic by attempt order. Round-robin, least-connections, passive health scoring, and circuit breakers are not yet implemented.
- Reload remains a graceful systemd generation recycle rather than an in-process atomic socket-preserving configuration swap.
- `[[listen.v4]]` / `[[listen.v6]]` is the beta.4 listener syntax. Existing flat `[[listen]]` configurations must be migrated before starting this build.
- Structured `[[site.proxy.multi.upstream]]` is still a beta configuration surface. The single `upstream = "..."` form remains supported.
