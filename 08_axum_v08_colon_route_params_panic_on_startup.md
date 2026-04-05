### Project
cortex

### Description
The app server panics immediately at startup — before serving a single request — because all route parameter segments are written with the old axum v0.7 colon syntax (`:param`) while the project depends on axum **0.8.8**, which no longer accepts that syntax. In axum ≥ 0.8 capture groups must be written as `{param}`. The router is constructed at startup in `routes()`, so the very first call to `.route()` that contains a `:param` segment triggers an unconditional panic inside axum, crashing the process.

### Error Message

```
thread 'main' (2237174) panicked at src/cortex-app-server/src/api/mod.rs:48:10:
Path segments must not start with `:`. For capture groups, use `{capture}`. If you
meant to literally match a segment starting with a colon, call `without_v07_checks`
on the router.
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
Aborted (core dumped)
```

### Debug Logs

```shell
$ cargo run -p cortex-app-server
...
thread 'main' panicked at src/cortex-app-server/src/api/mod.rs:48:10:
Path segments must not start with `:`. For capture groups, use `{capture}`. ...
Aborted (core dumped)
```

### System Information
- OS: Linux (Ubuntu 22.04)
- Rust toolchain: stable
- Cortex version: `cortex 0.0.7`
- axum version in workspace: **0.8.8** (`Cargo.toml` line 119)

### Screenshots

None.

### Steps to Reproduce

1. Clone the repository.
2. Run `cargo run -p cortex-app-server` (or any binary that calls `api::routes()`).
3. The process panics at startup before binding to any port.

### Expected Behavior

The server should start successfully and bind to its configured port, ready to handle HTTP requests.

### Actual Behavior

`routes()` in `src/cortex-app-server/src/api/mod.rs` is called at startup to build the axum `Router`. The **first** `.route()` call that contains a `:param` segment (line 48, `/proxy/:port`) causes axum 0.8 to panic unconditionally. The server never starts.

### Root Cause

`src/cortex-app-server/src/api/mod.rs` registers routes using the axum v0.7 colon syntax. The workspace `Cargo.toml` (line 119) declares `axum = "0.8.8"`, which dropped support for that syntax and requires `{capture}` instead. Every route with a dynamic segment is affected.

Affected lines and routes in `src/cortex-app-server/src/api/mod.rs`:

| Line | Current (broken) | Fixed |
|------|-------------------|-------|
| 48 | `/proxy/:port` | `/proxy/{port}` |
| 49 | `/proxy/:port/*path` | `/proxy/{port}/*path` |
| 53 | `/sessions/:id` | `/sessions/{id}` |
| 54 | `/sessions/:id` | `/sessions/{id}` |
| 55 | `/sessions/:id/messages` | `/sessions/{id}/messages` |
| 56 | `/sessions/:id/messages` | `/sessions/{id}/messages` |
| 59 | `/models/:id` | `/models/{id}` |
| 64 | `/tools/:name/execute` | `/tools/{name}/execute` |
| 83 | `/agents/:name` | `/agents/{name}` |
| 84 | `/agents/:name` | `/agents/{name}` |
| 85 | `/agents/:name` | `/agents/{name}` |
| 92 | `/stored-sessions/:id` | `/stored-sessions/{id}` |
| 96 | `/stored-sessions/:id` | `/stored-sessions/{id}` |
| 100 | `/stored-sessions/:id/history` | `/stored-sessions/{id}/history` |
| 105 | `/terminals/:id/logs` | `/terminals/{id}/logs` |

### Additional Context

axum 0.8 migration guide (https://github.com/tokio-rs/axum/blob/main/axum/CHANGELOG.md#0800-2024) explicitly states:

> Route parameters must now use `{param}` syntax instead of `:param`. Using the old syntax causes a panic at router construction time.

The fix is to rename every `:param` capture in every `.route()` call inside `routes()` to the `{param}` form. No handler code needs to change — axum's `Path` extractor works identically with the new syntax.

Example for the first two affected routes:

```rust
// Before (panics on axum 0.8)
.route("/proxy/:port",       get(proxy::proxy_to_port))
.route("/proxy/:port/*path", get(proxy::proxy_to_port_path))

// After (correct axum 0.8 syntax)
.route("/proxy/{port}",       get(proxy::proxy_to_port))
.route("/proxy/{port}/*path", get(proxy::proxy_to_port_path))
```
