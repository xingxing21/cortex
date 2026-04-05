### Project
cortex

### Description
In `validate_path_safety` inside `src/cortex-cli/src/uninstall_cmd.rs`, the sanity check that is supposed to allow Cortex-related paths is permanently bypassed due to a case mismatch. The guard at line 822 calls `.to_lowercase()` on the path string and then searches for the mixed-case literal `"Cortex"`, which can never match a fully-lowercase string. As a result the inner block is entered for **every** path, and the fallback binary check at line 828 (`name == "Cortex"`) also never matches because `name` is already lowercased. The uninstall command therefore bails out with *"Path does not appear to be Cortex-related"* for every file or directory it tries to delete — including config directories, data directories, and the `cortex` binary itself — making the entire uninstall operation silently non-functional on Linux and macOS.

### Error Message

```
Error: Path does not appear to be Cortex-related: /home/alice/.config/cortex
```

### Debug Logs

```shell
$ cortex uninstall --force
Preparing to uninstall Cortex...

Items to remove:
  • /home/alice/.local/bin/cortex (binary)
  • /home/alice/.config/cortex (config)
  • /home/alice/.local/share/cortex (data)

Error: Path does not appear to be Cortex-related: /home/alice/.local/bin/cortex
```

### System Information
- OS: Linux (Ubuntu 22.04)
- Cortex version: `cortex 0.0.7`

### Screenshots

None.

### Steps to Reproduce

1. Install Cortex on Linux or macOS.
2. Run `cortex uninstall --force`.
3. Observe that the command errors out immediately with "Path does not appear to be Cortex-related" for the very first path it tries to remove.

### Expected Behavior

`validate_path_safety` should pass for all legitimate Cortex installation paths (e.g. `~/.config/cortex`, `~/.local/bin/cortex`). The `"..."` bail-out should only be reached for genuinely unrelated paths.

### Actual Behavior

The sanity check at line 822 is never satisfied:

```rust
// to_lowercase() converts the haystack to all-lowercase, then searches for
// mixed-case "Cortex" — this can NEVER match, so the body is always entered.
if !path_str.to_lowercase().contains("Cortex") {
    let is_binary = path
        .file_name()
        .map(|n| {
            let name = n.to_string_lossy().to_lowercase();
            // name is already lowercase, so "Cortex" never equals name.
            // Only "cortex.exe" / "cortex.old" accidentally work because
            // they are already lowercase literals.
            name == "Cortex" || name == "cortex.exe" || name == "cortex.old"
        })
        .unwrap_or(false);

    if !is_binary {
        bail!(
            "Path does not appear to be Cortex-related: {}",
            path.display()
        );
    }
}
```

Because the guard condition is always `true` (the body is always entered) and the binary check always returns `false` for the plain `cortex` binary, every call to `validate_path_safety` for a standard installation path bails with the error above.

### Additional Context

Code evidence in `src/cortex-cli/src/uninstall_cmd.rs`, lines 822–838.

The fix is to lowercase the needle to match the lowercased haystack, and likewise lowercase the binary-name comparison:

```rust
// Line 822 – use lowercase needle
if !path_str.to_lowercase().contains("cortex") {
    let is_binary = path
        .file_name()
        .map(|n| {
            let name = n.to_string_lossy().to_lowercase();
            // Line 828 – name is lowercase, so compare with lowercase literals
            name == "cortex" || name == "cortex.exe" || name == "cortex.old"
        })
        .unwrap_or(false);

    if !is_binary {
        bail!(
            "Path does not appear to be Cortex-related: {}",
            path.display()
        );
    }
}
```
