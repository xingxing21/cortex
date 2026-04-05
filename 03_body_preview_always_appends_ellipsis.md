### Project
cortex

### Description
In `print_event_summary` inside `src/cortex-cli/src/github_cmd.rs`, the dry-run body preview for `IssueComment` events unconditionally appends `"..."` to the output string regardless of whether the body was actually truncated. A comment body of 10 characters will display as `"Hello world..."` even though nothing was cut off, misleading the user into thinking the output is incomplete.

### Error Message

None — the output is visually wrong but does not error.

### Debug Logs

```shell
$ cortex github run --event issue_comment --dry-run \
    --token ghp_xxx --repository owner/repo --event-path /tmp/short_comment.json
Dry run mode - not executing

Event: Issue Comment
  Action: created
  Issue #: 7
  Author: alice
  Body preview: fix typo...        # <-- "..." appended even though body is only 8 chars
```

### System Information
- OS: Linux (Ubuntu 22.04)
- Cortex version: `cortex 0.0.7`

### Screenshots

None.

### Steps to Reproduce

1. Create a minimal issue-comment event JSON payload whose `body` field is shorter than 100 characters.
2. Run `cortex github run --event issue_comment --dry-run --token ... --repository ... --event-path ...`.
3. Observe that the "Body preview" line ends with `"..."` even though no truncation occurred.

### Expected Behavior

The `"..."` suffix should only be appended when the comment body exceeds 100 characters (i.e., when it was actually truncated). Short bodies should be printed as-is.

### Actual Behavior

The format string hardcodes `"..."` unconditionally:

```rust
println!(
    "  Body preview: {}...",  // "..." always appended
    comment.body.chars().take(100).collect::<String>()
);
```

This means even a one-word body like `"LGTM"` is displayed as `"LGTM..."`.

### Additional Context

Code evidence in `src/cortex-cli/src/github_cmd.rs`, lines 340–343:

```rust
println!(
    "  Body preview: {}...",
    comment.body.chars().take(100).collect::<String>()
);
```

The fix is to check the actual character count before deciding whether to append `"..."`:

```rust
let chars: Vec<char> = comment.body.chars().collect();
let (preview, ellipsis) = if chars.len() > 100 {
    (chars[..100].iter().collect::<String>(), "...")
} else {
    (chars.iter().collect::<String>(), "")
};
println!("  Body preview: {}{}", preview, ellipsis);
```
