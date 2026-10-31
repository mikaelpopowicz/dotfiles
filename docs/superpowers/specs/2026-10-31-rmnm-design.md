# rmnm — cascade `node_modules` cleanup (design)

Date: 2026-10-31  
Status: implemented

## Goal

Provide a polished interactive zsh function to find and delete `node_modules` directories under a root path, with dry-run support and a clear final report (paths + space reclaimed).

## Placement

- Single entry point: function `rmnm` in `shell/.functions`
- Private helpers: `_rmnm_*` in the same file
- Style aligned with existing gum-based Docker helpers (`dcu`, `dcs`, …)
- Dependency: `gum` (already used in this repo). Fail fast with a clear message if missing.

## CLI contract

```bash
rmnm                  # root = $PWD
rmnm ~/Code           # root = given directory
rmnm -n               # dry-run via flag
rmnm --dry-run ~/Code
rmnm -h | --help      # short styled help
```

Rules:

- First positional argument = root directory; default `.`
- Reject missing / non-directory roots with a clear error and non-zero exit
- Unknown flags → error + hint to `-h`
- Out of scope for v1: non-interactive / CI mode, exclusion globs, `fd`

## Discovery

- Use `find` (no `-L`): do not follow symlinks
- Match `-type d -name node_modules`
- **Prune**: do not descend into a found `node_modules` (one folder per project tree)
- Measure sizes numerically (e.g. `du -sk`) so sort and totals are correct; format for display (human units)
- Present list sorted by size descending
- Handle paths with spaces safely (NUL-delimited pipeline or equivalent)
- Grand total = sum of selected (or found) byte/KiB sizes, then human-formatted once — never by parsing `12M`-style strings

## TUI flow

Polish matters: consistent headers, badges, aligned columns, spinner feedback, cancel = no side effects.

1. **Header** — title `rmnm`, scanned root, `DRY-RUN` badge when dry-run is active (flag or menu)
2. **Scan** — `gum spin` while finding dirs and measuring sizes
3. **Summary** — `N found · X total` via `gum style`
4. **Mode menu** (skipped as a choice if `-n`/`--dry-run` already set; still show dry-run badge):
   - Delete
   - Dry-run
   - Cancel → exit 0, nothing deleted
5. **Scope menu**:
   - All
   - Select… → `gum filter --no-limit` with lines formatted as `SIZE<pad>PATH` (e.g. `12M   ./app/node_modules`)
6. **Confirm** — `gum confirm` with count + total size before destructive delete  
   Dry-run: skip destructive confirm; proceed to the same report shape labeled as would-delete / would-free
7. **Execute** — `gum spin` during `rm -rf` (delete mode only)
8. **Final report** (always):
   - Each path with its size
   - Total space recovered (or “would free” in dry-run)
   - Counts: succeeded / failed if any `rm` fails (continue on partial failure)

Empty result: friendly message, exit 0.

## Internals (helpers)

| Helper | Role |
|--------|------|
| `_rmnm_usage` | Styled help |
| `_rmnm_find` | Discover pruned `node_modules` paths |
| `_rmnm_sizes` | Build `size\tpath` rows, total, sort by size desc |
| `_rmnm_pick` | Mode + scope menus, return selected paths |
| `_rmnm_delete` | `rm -rf` selected paths; track failures |
| `_rmnm_report` | Styled final report (delete or dry-run wording) |

Main `rmnm` orchestrates parse → scan → pick → delete-or-noop → report.

## Error handling

- Missing `gum` → message on stderr, exit 1
- Invalid root → message, exit 1
- Cancel / Esc at any gum prompt → exit 0, no deletions
- Partial `rm` failures → continue; report marks failures; exit non-zero if any failed

## Out of scope (v1)

- Non-interactive flags beyond `--dry-run` (e.g. `--yes`, `--json`)
- Searching nested `node_modules` inside another `node_modules`
- Size caching, parallel `du`, `fd`/`dust` integration
- Deleting other artifact dirs (`vendor`, `.next`, etc.)

## Success criteria

- From a root with several projects, user can dry-run or delete all / a subset in under a few interactions
- Sizes visible before any destructive action
- Final report always shows what happened and how much space was (or would be) freed
- UX feels consistent with existing gum helpers, but more visually finished (header, badge, columns, report)
