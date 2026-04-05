### Project
cortex

### Description
`cortex lock remove` silently fails to unlock a session when the user provides a valid 8-character hex prefix, even though `cortex lock check` and `cortex lock list` both accept and resolve the same prefix correctly. The remove operation uses exact-string matching only, so a session that was locked under its full UUID can never be unlocked by prefix — the command exits with "None of the specified sessions are locked." despite the session being demonstrably locked.

### Error Message

```
$ cortex lock 550e8400     # locks by 8-char prefix
Locked 1 session(s).

$ cortex lock check 550e8400
Session '550e8400' is LOCKED.

$ cortex lock remove 550e8400
Error: None of the specified sessions are locked.
```

(Or vice-versa: lock by full UUID, attempt to remove by 8-char prefix.)

### Debug Logs

```
None of the specified sessions are locked.
```

No stack trace is emitted; the function bails with a user-visible error via `anyhow::bail!`.

### System Information
- OS: Linux
- Rust toolchain: stable
- Cortex version: `cortex 0.0.7`

### Screenshots

None.

### Steps to Reproduce

**Scenario A — lock by full UUID, remove by prefix:**
1. Run `cortex lock add 550e8400-e29b-41d4-a716-446655440000`
2. Run `cortex lock check 550e8400` → correctly prints "Session '550e8400' is LOCKED."
3. Run `cortex lock remove 550e8400` → **incorrectly** prints "None of the specified sessions are locked."

**Scenario B — lock by prefix, remove by prefix:**
1. Run `cortex lock add 550e8400`
2. Run `cortex lock remove 550e8400` → works (exact match hits)
3. Run `cortex lock add 550e8400` again
4. Run `cortex lock remove 550e8400-e29b-41d4-a716-446655440000` → **incorrectly** prints "None of the specified sessions are locked."

### Expected Behavior

`cortex lock remove <id>` should resolve the provided ID the same way every other lock operation does: match any stored session whose `session_id` either equals the argument exactly **or** starts with the 8-character prefix supplied by the user. Unlock should succeed in all cases where `cortex lock check <id>` reports the session as locked.

### Actual Behavior

`run_remove` builds a `HashSet<String>` from the raw arguments and checks `session_ids.contains(&e.session_id)` — a plain equality check. It does not apply the 8-character prefix resolution used by `is_session_locked` and `run_check`. Sessions locked under a full UUID cannot be removed by prefix, and sessions locked under a prefix cannot be removed by full UUID.

### Root Cause

`src/cortex-cli/src/lock_cmd.rs`, function `run_remove` (lines 256–292):

```rust
// run_remove — exact match only (BUG)
let session_ids: HashSet<String> = args.session_ids.iter().cloned().collect();
let to_remove: Vec<_> = lock_file
    .locked_sessions
    .iter()
    .filter(|e| session_ids.contains(&e.session_id))   // ← exact match only
    .map(|e| e.session_id.clone())
    .collect();
```

Compare with `is_session_locked` and `run_check`, which both apply prefix resolution:

```rust
// is_session_locked — prefix-aware (correct)
lock_file.locked_sessions.iter().any(|entry| {
    entry.session_id == session_id
        || session_id.starts_with(&entry.session_id[..8.min(entry.session_id.len())])
})
```

The same discrepancy exists in `run_add`'s duplicate-detection check (line 225,
`existing_ids.contains(session_id)`), which means the same logical session can be
locked twice if once provided as a full UUID and once as an 8-char prefix — creating
duplicate entries that both resolve as locked but only one of which can be removed.

### Fix

Apply the same prefix-resolution logic used by `is_session_locked` inside `run_remove` when
filtering `locked_sessions`:

```rust
// After fix — prefix-aware removal
let to_remove: Vec<_> = lock_file
    .locked_sessions
    .iter()
    .filter(|e| {
        session_ids.iter().any(|id| {
            e.session_id == *id
                || e.session_id.starts_with(&id[..8.min(id.len())])
                || id.starts_with(&e.session_id[..8.min(e.session_id.len())])
        })
    })
    .map(|e| e.session_id.clone())
    .collect();
```

Apply the same fix to the duplicate-detection block in `run_add` for consistency.

### Additional Context

`validate_session_id` (line 100) explicitly allows 8-character hex prefixes as valid input, so users are guided to use them. The broken behavior in `run_remove` directly contradicts that documented interface. The `lock check` and `lock remove` subcommands must use the same resolution semantics, otherwise sessions can become permanently unlockable via the CLI without manual edits to `~/.cortex/session_locks.json`.
