### Project
cortex

### Description
`cortex github run` advertises in its error messages that missing arguments can be supplied via `GITHUB_TOKEN`, `GITHUB_REPOSITORY`, and `GITHUB_EVENT_PATH` environment variables, but the function `run_github_agent` never actually reads those environment variables. It only unwraps the `args.token`, `args.repository`, and `args.event_path` fields (which are populated solely from CLI flags `--token`, `--repository`, and `--event-path`). Setting the environment variables has no effect; the command always exits with an error if the flags are omitted.

### Error Message

```shell
Error: GitHub token required. Set GITHUB_TOKEN env var or use --token
```

### Debug Logs

```shell
$ export GITHUB_TOKEN=ghp_xxxxxxxxxxxx
$ export GITHUB_REPOSITORY=owner/repo
$ export GITHUB_EVENT_PATH=/tmp/event.json
$ cortex github run --event issues
Error: GitHub token required. Set GITHUB_TOKEN env var or use --token
```

### System Information
- OS: Linux (Ubuntu 22.04)
- Cortex version: `cortex 0.0.7`

### Screenshots

None.

### Steps to Reproduce

1. Export the standard GitHub Actions environment variables:
   ```shell
   export GITHUB_TOKEN=ghp_xxxxxxxxxxxx
   export GITHUB_REPOSITORY=owner/repo
   export GITHUB_EVENT_PATH=/tmp/event.json
   ```
2. Run `cortex github run --event issues` (omitting `--token`, `--repository`, `--event-path`).
3. Observe the error despite the env vars being set.

### Expected Behavior

When `--token`, `--repository`, or `--event-path` are not provided as CLI flags, `run_github_agent` should fall back to reading `GITHUB_TOKEN`, `GITHUB_REPOSITORY`, and `GITHUB_EVENT_PATH` respectively from the environment, matching the documented behavior in the error messages.

### Actual Behavior

The function only checks `args.token`, `args.repository`, and `args.event_path` (CLI flags). It never calls `std::env::var("GITHUB_TOKEN")` or any equivalent, so setting the environment variables has no effect. The command always fails if the flags are not provided explicitly.

### Additional Context

Code evidence in `src/cortex-cli/src/github_cmd.rs`:

```rust
// Line 263-278
let token = args.token.ok_or_else(|| {
    anyhow::anyhow!("GitHub token required. Set GITHUB_TOKEN env var or use --token")
    // ^^^ Mentions env var but never reads it
})?;

let repository = args.repository.ok_or_else(|| {
    anyhow::anyhow!(
        "GitHub repository required. Set GITHUB_REPOSITORY env var or use --repository"
        // ^^^ Same problem
    )
})?;

let event_path = args.event_path.ok_or_else(|| {
    anyhow::anyhow!(
        "Event payload path required. Set GITHUB_EVENT_PATH env var or use --event-path"
        // ^^^ Same problem
    )
})?;
```

The fix is to change each `ok_or_else` to first attempt `std::env::var(...)` before returning an error, e.g.:

```rust
let token = args.token
    .or_else(|| std::env::var("GITHUB_TOKEN").ok())
    .ok_or_else(|| anyhow::anyhow!("GitHub token required. Set GITHUB_TOKEN env var or use --token"))?;
```
