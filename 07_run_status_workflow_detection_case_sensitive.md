### Project
cortex

### Description
In `run_status` inside `src/cortex-cli/src/github_cmd.rs`, the check that detects whether a Cortex workflow is installed uses a case-sensitive `content.contains("Cortex")` search. When a workflow was installed with `cortex github install --workflow-name cortex` (lowercase — the default for the `uninstall` and `update` sub-commands), the generated YAML file uses the lowercase name throughout and may not contain the exact string `"Cortex"`. The detection therefore returns `false`, `status.workflow_installed` stays `false`, and `cortex github status` incorrectly reports that no Cortex workflow is found — then calls `std::process::exit(1)`, making CI pipelines that call `cortex github status` fail spuriously.

### Error Message

```
Cortex workflow not found.
   Run `cortex github install` to set up GitHub Actions.
```
(with exit code 1)

### Debug Logs

```shell
$ cortex github install --workflow-name cortex
GitHub Actions workflow installed!
   Location: .github/workflows/cortex.yml

$ cortex github status
GitHub Actions Status
========================================

Cortex workflow not found.           # <-- false negative: workflow IS installed
   Run `cortex github install` to set up GitHub Actions.

$ echo $?
1                                     # <-- exit code 1 breaks CI
```

### System Information
- OS: Linux (Ubuntu 22.04)
- Cortex version: `cortex 0.0.7`

### Screenshots

None.

### Steps to Reproduce

1. Run `cortex github install --workflow-name cortex` (lowercase — which is the default for both `github uninstall` and `github update`).
2. Run `cortex github status`.
3. Observe the false-negative "Cortex workflow not found" error and exit code 1.

### Expected Behavior

`run_status` should detect the Cortex workflow regardless of the capitalisation of its name, and should report `workflow_installed = true` for any workflow file whose content refers to Cortex in any capitalisation.

### Actual Behavior

The detection is case-sensitive:

```rust
// src/cortex-cli/src/github_cmd.rs, line 601
if content.contains("Cortex") {   // exact-case check — misses lowercase "cortex"
    status.workflow_installed = true;
    ...
}
```

Compare this with the `run_uninstall` function in the same file (line 698), which correctly handles both cases:

```rust
// github_cmd.rs, line 698 — uninstall does it right
&& (content.contains("Cortex") || content.contains("cortex"))
```

`run_status` only checks for `"Cortex"` (capital C) and therefore misses any workflow installed with a lowercase name.

### Additional Context

Code evidence in `src/cortex-cli/src/github_cmd.rs`, line 601.

There is also an inconsistency in the default `--workflow-name` values across sub-commands:
- `cortex github install` defaults to `"Cortex"` (capital C, line 66)
- `cortex github uninstall` defaults to `"cortex"` (lowercase, line 118)
- `cortex github update` defaults to `"cortex"` (lowercase, line 134)

Because of this mismatch, a user who runs `cortex github install` followed by `cortex github status` will see the correct result, but a user who re-installs with the lowercase default of `uninstall`/`update` will hit the false negative.

The fix is to make the status check case-insensitive, matching `run_uninstall`:

```rust
// Line 601 – check for either capitalisation
if content.to_lowercase().contains("cortex") {
    status.workflow_installed = true;
    ...
}
```
