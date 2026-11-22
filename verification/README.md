# Dual-container differential verification

- `reference/` — pinned Go baseline (gnet v2.10.0, commit 58b59a1) run in a container.
- `candidate/` — Rust build of this workspace run in a second container.
- Both run against the same scenario corpus under `verification/scenarios/`;
  outputs are compared field-by-field and evidence lands in `verification/evidence/*.json`.

Constraints (per AINative methodology): tmpfs one-shot data, no published ports,
`cap_drop: ALL`, full cleanup after each run.
