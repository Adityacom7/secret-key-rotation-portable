# AGENTS.md — Secret Key Rotation Tool (portable)

For Codex and any agent that reads `AGENTS.md`. Full context/rules are in
`CLAUDE.md` and `.kiro/steering/*.md`. Read them before editing.

Summary: a localhost-only web tool to safely add/update keys in AWS Secrets
Manager (fetch → merge-in-memory-with-json → put whole blob). Built to avoid the
PEM/`\n` corruption that shell+jq scripts cause. This repo ships ONE runnable
file, `secret-key-rotation.sh`, which writes its app into `skr-app/` on run.

Hard rules (see CLAUDE.md for detail):
- Localhost only; never bind `0.0.0.0`.
- JSON engine only for secret values (Python `json`; `jq` only as a verifier).
  No shell/sed/echo/inline-jq JSON construction.
- Whole-blob merge; writes via temp file; never inline secret args.
- No hardcoded accounts, org names, usernames, or local paths — discover at
  runtime. Keep it portable.
- Reads/previews/discovery are read-only; only Apply and Rollback-confirm write,
  after a UI confirmation showing account + ARN.
- Keep the dual (Python+jq) key count, the `after − added == before` check, and
  the scan→apply drift check.

The real source lives embedded (base64) inside `secret-key-rotation.sh`; the
`skr-app/` folder is regenerated on each run, so do not treat it as source.
