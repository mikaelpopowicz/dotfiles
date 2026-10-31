# rmart family — cascade artifact cleanup (design)

Date: 2026-10-31  
Status: approved for implementation  
Extends: `2026-10-31-rmnm-design.md`

## Goal

Generalize `rmnm` into a shared core so Composer (`vendor`) and Cargo (`target`) get the same polished TUI, with thin command wrappers.

## Commands

| Command | Directory | Label |
|---------|-----------|-------|
| `rmnm` | `node_modules` | node_modules |
| `rmcomposer` | `vendor` | vendor |
| `rmcargo` | `target` | target |

CLI identical to `rmnm`: `[-n|--dry-run] [-h|--help] [root]`

## Architecture

- Private core: `_rmart <cmd> <dirname> <label> [args…]`
- Helpers: `_rmart_human_kib`, `_rmart_scan`, `_rmart_header`, `_rmart_report`, …
- Wrappers call `_rmart` only
- Same TUI flow, prune find, numeric sizes, dry-run flag+menu, final report

## Out of scope (v1)

- Unified multi-ecosystem picker (Node + PHP + Rust in one run) — future; core stays open to it

## Success

- `rmcomposer` / `rmcargo` behave like `rmnm` on their directory names
- No duplicated TUI logic across the three commands
