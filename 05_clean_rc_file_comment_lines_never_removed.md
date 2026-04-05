### Project
cortex

### Description
In `clean_rc_file` inside `src/cortex-cli/src/uninstall_cmd.rs`, comment-only lines that were written by the Cortex installer (e.g. `# Cortex bash completions`) are never cleaned up from shell rc files during uninstall. The fourth OR-arm of the removal condition calls `.to_lowercase()` on the line and then checks `.contains("Cortex")` — a mixed-case needle — against an all-lowercase string, which can never match. As a result `cortex uninstall` leaves stale comment lines in `~/.bashrc`, `~/.zshrc`, `~/.bash_profile`, and `~/.profile` even after a successful uninstall.

### Error Message

None — the rc files are left in a dirty state but no error is reported.

### Debug Logs

```shell
$ cortex uninstall --force
...
  Cleaned: /home/alice/.bashrc     # <-- file written, but Cortex comment lines remain

$ grep -n Cortex ~/.bashrc
42: # Cortex bash completions      # <-- stale line left behind
```

### System Information
- OS: Linux (Ubuntu 22.04)
- Cortex version: `cortex 0.0.7`

### Screenshots

None.

### Steps to Reproduce

1. Install Cortex on Linux or macOS (the installer adds a `# Cortex bash completions` comment and a `source` line to `~/.bashrc`).
2. Run `cortex uninstall --force`.
3. Inspect `~/.bashrc` — the `source` / `eval` lines are removed, but the `# Cortex bash completions` comment line remains.

### Expected Behavior

`clean_rc_file` should remove both the functional shell lines **and** any adjacent Cortex comment lines (lines whose only content is a `#`-prefixed Cortex annotation).

### Actual Behavior

The fourth OR-arm of the removal predicate is always `false`:

```rust
let should_remove = patterns.iter().any(|p| {
    line.contains(p)
        && (line.contains("completion")
            || line.contains("source")
            || line.contains("eval")
            // to_lowercase() produces an all-lowercase string;
            // contains("Cortex") with a mixed-case needle never matches.
            || line.trim().starts_with('#') && line.to_lowercase().contains("Cortex"))
});
```

Because the last arm is always `false`, a line like `# Cortex bash completions` is evaluated only against the first three conditions (`completion`, `source`, `eval`). The line does happen to contain `"completion"`, so in this specific example it is accidentally removed — but a pure annotation line such as `# Added by Cortex installer` that lacks those keywords is silently kept.

### Additional Context

Code evidence in `src/cortex-cli/src/uninstall_cmd.rs`, line 907.

The fix is to lowercase the needle so it matches the lowercased line:

```rust
// Line 907 – use lowercase needle "cortex" instead of "Cortex"
|| line.trim().starts_with('#') && line.to_lowercase().contains("cortex")
```
