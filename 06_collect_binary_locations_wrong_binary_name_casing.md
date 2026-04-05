### Project
cortex

### Description
In `collect_binary_locations` inside `src/cortex-cli/src/uninstall_cmd.rs`, two of the hardcoded binary search paths use the capitalised filename `"Cortex"` instead of the lowercase `"cortex"` that the binary actually has on Linux and macOS. On a case-sensitive filesystem (Linux ext4, btrfs, etc.) `~/.local/bin/Cortex` and `~/.cargo/bin/Cortex` never exist, so `.exists()` returns `false` for both entries and neither path is added to the removal list. As a result `cortex uninstall` silently leaves the installed binary behind in `~/.local/bin` and/or `~/.cargo/bin`.

### Error Message

None — the binary is simply left on disk with no warning.

### Debug Logs

```shell
$ cortex uninstall --force
...
✓ Cortex CLI has been successfully uninstalled.

$ which cortex
/home/alice/.local/bin/cortex      # <-- binary is still present
```

### System Information
- OS: Linux (Ubuntu 22.04, ext4 — case-sensitive filesystem)
- Cortex version: `cortex 0.0.7`

### Screenshots

None.

### Steps to Reproduce

1. Install Cortex on Linux into `~/.local/bin/cortex` (the default installation path).
2. Run `cortex uninstall --force`.
3. Observe that the binary remains at `~/.local/bin/cortex` after the uninstall completes successfully.

### Expected Behavior

`collect_binary_locations` should probe `~/.local/bin/cortex` (lowercase) and `~/.cargo/bin/cortex` (lowercase) so that `uninstall` correctly detects and removes the binary on Linux.

### Actual Behavior

The two paths in the hardcoded binary-location list use the wrong capitalisation:

```rust
// src/cortex-cli/src/uninstall_cmd.rs, lines 366–367
let binary_locations = [
    home_dir.join(".local").join("bin").join("Cortex"),   // ← capital C — path never exists on Linux
    home_dir.join(".cargo").join("bin").join("Cortex"),   // ← capital C — path never exists on Linux
    #[cfg(not(target_os = "windows"))]
    PathBuf::from("/usr/local/bin/cortex"),               // ← correctly lowercase
    #[cfg(target_os = "windows")]
    home_dir.join(".cargo").join("bin").join("cortex.exe"),
];
```

Because `path.exists()` is `false` for both capitalised paths, neither entry is added to the removal list and the binary is never deleted.

### Additional Context

Code evidence in `src/cortex-cli/src/uninstall_cmd.rs`, lines 365–372.

Note the inconsistency: the non-Windows system path on line 369 already uses the correct lowercase `"cortex"`, as does the Windows path `"cortex.exe"` on line 371 — only the two home-directory paths on lines 366–367 are wrong.

The fix is to lowercase both filenames:

```rust
let binary_locations = [
    home_dir.join(".local").join("bin").join("cortex"),   // lowercase — matches actual binary
    home_dir.join(".cargo").join("bin").join("cortex"),   // lowercase — matches actual binary
    #[cfg(not(target_os = "windows"))]
    PathBuf::from("/usr/local/bin/cortex"),
    #[cfg(target_os = "windows")]
    home_dir.join(".cargo").join("bin").join("cortex.exe"),
];
```
