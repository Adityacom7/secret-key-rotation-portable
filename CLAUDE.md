# CLAUDE.md — Secret Key Rotation Tool (portable)

Context for any AI agent (Claude Code, Kiro, Codex) working in this repo. Read
this plus `.kiro/steering/*.md` before changing anything.

## What this is
A **local-only** web tool (localhost backend + single HTML page) to safely add /
update keys across many AWS Secrets Manager secrets, with verification, local
backups, and rollback. Built because older scripts corrupted values containing
line breaks / escapes (PEM keys with `\n`).

## Repo shape (portable, single file)
This repo ships ONE runnable file: `secret-key-rotation.sh`. On run it writes
its app into `skr-app/`:
- `skr-app/server.py`  — Python `http.server` on `127.0.0.1` only; uses `boto3`.
  Holds ALL logic: account discovery, scan+backup, preview, apply, rollback,
  jq verify.
- `skr-app/index.html` — one page; calls local `/api/*` endpoints.
- `skr-app/.venv`, `skr-app/backups/`, `skr-app/accounts.cache.json` — runtime,
  git-ignored.

**The source of truth is the embedded code inside `secret-key-rotation.sh`**
(base64 of server.py + index.html). To change behavior, edit the embedded
sources (or regenerate the script from a source copy with its build step) — do
not hand-edit only `skr-app/`, since that folder is recreated on each run.

## Non-negotiable rules
1. **Localhost only.** Never bind `0.0.0.0`. The server refuses non-loopback.
2. **JSON engine only** for secret values: Python `json.loads`/`json.dumps`.
   `jq` is read-only verification. NEVER build JSON with shell strings, `sed`,
   `echo`, or inline `jq` construction — that is the corruption this tool fixes.
3. **Whole-blob merge:** GetSecretValue → parse → merge in memory →
   PutSecretValue the full merged JSON.
4. **Writes via temp file**, never inline CLI args; never echo secret values.
5. **No hardcoded accounts / org names / usernames / absolute local paths.**
   Discover accounts at runtime. Keep it portable.
6. **Reads/previews/discovery are read-only.** Only Apply and Rollback-confirm
   write, after a UI confirmation showing account + ARN.
7. **Count integrity.** Keep the dual (Python+jq) count, the
   `keys_after − added == keys_before` check, and the scan→apply drift check.
8. Default region `ap-south-1`; override via env `SKR_DEFAULT_REGION`.
   Port is time-based (`HHMM` + `7`); override via env `PORT`.

## Endpoints (server.py)
- `GET  /api/saved-accounts`    — load per-machine account cache (no AWS).
- `POST /api/discover-accounts` — scan machine, resolve account ids, save cache.
- `POST /api/check-identity`    — sts get-caller-identity for a profile.
- `POST /api/check-permissions` — non-mutating IAM simulate (read/describe/put).
- `POST /api/scan-backup`       — read-only scan + local backup, dual key count.
- `POST /api/jq-verify`         — jq re-count on local backups (no AWS).
- `POST /api/preview`           — dry-run add/replace diff + safety math.
- `POST /api/apply`             — WRITE: merge + put (gated, drift-checked).
- `POST /api/rollback-preview`  — dry-run restore diff.
- `POST /api/rollback-apply`    — WRITE: restore full backup snapshot.

## Backups layout (under skr-app/backups/)
`<account>_<ts>/`            — scan runs (rollback restore source)
`<account>_<ts>_APPLY/`      — pre-write snapshots
`<account>_<ts>_ROLLBACK/`   — pre-rollback snapshots
Each backup file is named by its secret path and stores the raw `secret_string`.

## Testing
- Test against a NON-prod profile. Previews/scans/discovery are read-only.
- Real writes (apply/rollback) should be triggered by the human operator.
- Never run writes against a prod account without explicit instruction.
