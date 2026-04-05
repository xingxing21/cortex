### Project
cortex

### Description
In `handle_issues` inside `src/cortex-cli/src/github_cmd.rs`, the `is_cortex_related` check calls `.to_lowercase()` on the issue title, body, and labels, then searches the result for the string `"Cortex"` (capital C). Because `to_lowercase()` converts all characters to lowercase, the resulting string can never contain the capital-C substring `"Cortex"`. Consequently `is_cortex_related` is always `false`, the greeting comment is never posted, and the whole issue-automation feature is silently broken for every newly opened issue.

### Error Message

None — the function silently skips posting the greeting with no output.

### Debug Logs

```shell
$ cortex github run --event issues --token ghp_xxx --repository owner/repo --event-path /tmp/issue_opened.json
Processing issue #42
   Title: Cortex crashes on startup
   Action: opened
# <-- no greeting comment is ever posted
```

### System Information
- OS: Linux (Ubuntu 22.04)
- Cortex version: `cortex 0.0.7`

### Screenshots

None.

### Steps to Reproduce

1. Configure a GitHub Actions workflow with `cortex github install`.
2. Open a new issue whose title or body contains the word "Cortex" (any capitalisation).
3. Observe that no welcome greeting comment is posted on the issue.

### Expected Behavior

When a new issue mentioning "cortex" (case-insensitive) is opened, `handle_issues` should post a greeting comment to welcome the reporter and explain available commands.

### Actual Behavior

`is_cortex_related` is always `false` because the code compares a lowercased string against a mixed-case needle:

```rust
let is_cortex_related = issue.title.to_lowercase().contains("Cortex")  // never true
    || issue.body.to_lowercase().contains("Cortex")                     // never true
    || issue.labels.iter().any(|l| l.to_lowercase().contains("Cortex")); // never true
```

No greeting is ever posted regardless of the issue content.

### Additional Context

Code evidence in `src/cortex-cli/src/github_cmd.rs`, lines 526–531:

```rust
let is_cortex_related = issue.title.to_lowercase().contains("Cortex")
    || issue.body.to_lowercase().contains("Cortex")
    || issue
        .labels
        .iter()
        .any(|l| l.to_lowercase().contains("Cortex"));
```

The needle passed to `.contains()` must be lowercase to match the lowercased haystack. The fix is to change all three needles from `"Cortex"` to `"cortex"`:

```rust
let is_cortex_related = issue.title.to_lowercase().contains("cortex")
    || issue.body.to_lowercase().contains("cortex")
    || issue.labels.iter().any(|l| l.to_lowercase().contains("cortex"));
```
