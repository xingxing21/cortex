# Cortex Project Optimization Plan

## TL;DR
> **Quick Summary**: Comprehensive refactoring to eliminate code duplicates, consolidate types, optimize compilation, and improve codebase structure.

## Progress

### ✅ COMPLETED (Phase 1)
| Task | Files | Status |
|------|-------|--------|
| `default_true()` consolidation | 21 files | ✅ Done |
| `timestamp_now()` consolidation | 20 files | ✅ Done |
| `read_file_with_encoding()` consolidation | 2 files | ✅ Done |
| Created `cortex-common/src/serde_helpers.rs` | 1 file | ✅ Done |
| Added `timestamp_now()` to cortex-common | 1 file | ✅ Done |

### 🔄 IN PROGRESS (Phase 2)
| Task | Agent | Status |
|------|-------|--------|
| Type consolidation (SessionInfo, AgentConfig, Hunk, SandboxPolicy) | bg_74ddd305 | Running |
| tokio "full" optimization (9 crates) | bg_cac5bcea | Running |
| `format_duration()` consolidation | bg_a980bd63 | Running |
| `truncate*()` consolidation | bg_1bd1c787 | Running |

### 📋 PENDING (Phase 3)
| Task | Priority |
|------|----------|
| Merge `cortex-utils/*` crates | Medium |
| Consolidate `glob_match()` (8 duplicates) | Medium |
| Consolidate `parse_frontmatter()` (4 duplicates) | Medium |
| Consolidate `expand_tilde()` (3 duplicates) | Low |

## Summary Statistics

### Before Optimization
- Total crates: 62
- Duplicate `default_true()`: 21 occurrences
- Duplicate `timestamp_now()`: 20 occurrences
- Duplicate types: 20 definitions
- `tokio full` usage: 9 crates

### After Phase 1
- Duplicate `default_true()`: 1 (in cortex-common)
- Duplicate `timestamp_now()`: 1 (in cortex-common)
- Compilation: ✅ All modified crates pass `cargo check`

## Key Decisions
1. `cortex-common` as the canonical location for shared utilities
2. `cortex-protocol` as the canonical location for shared types
3. Use `use cortex_common::function` pattern everywhere

## Next Steps
1. Wait for background agents to complete
2. Run full `cargo check --workspace`
3. Run `cargo test --workspace`
4. Create summary commit
