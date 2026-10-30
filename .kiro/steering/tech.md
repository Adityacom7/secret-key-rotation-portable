# Tech — Secret Key Rotation Tool (portable)

## Stack
- **Python 3** standard library `http.server` (no web framework).
- **boto3** — the only pip dependency; auto-installed into a local `.venv`.
- **jq** — optional, used ONLY as an independent key-count verifier.
- Plain HTML/CSS/vanilla JS for the UI (no build step, no npm).

## Running
- `./secret-key-rotation.sh` — writes the app into `skr-app/`, creates
  `skr-app/.venv`, installs `boto3` if missing, then launches on `127.0.0.1`.
- Port is time-based: 24h `HHMM` with a `7` appended (e.g. 17:13 → 17137),
  leading zero dropped. Override by exporting `PORT`.
- The URL is printed in the terminal on startup.

## Hard technical rules
1. **Localhost only.** The server refuses to bind to a non-loopback host.
2. **JSON engine only** for secret values: Python `json.loads` / `json.dumps`.
   `jq` is read-only verification. NEVER build JSON with shell strings, `sed`,
   `echo`, or inline `jq` construction — that is the corruption this tool fixes.
3. **Whole-blob merge:** GetSecretValue → parse → merge in memory →
   PutSecretValue the full merged JSON.
4. **Writes via temp file**, never inline CLI args; never echo secret values.
5. **No hardcoded accounts / org names / usernames / absolute local paths.**
   Account labels and the prod flag are inferred at runtime from the profile
   name / ARN. Discovery reads whatever profiles exist on the machine.
6. Default region `ap-south-1`; override with env `SKR_DEFAULT_REGION`.

## AWS calls used
- Read-only: `sts:GetCallerIdentity`, `secretsmanager:GetSecretValue`,
  `secretsmanager:DescribeSecret`, `iam:SimulatePrincipalPolicy`.
- Write (only on confirmed Apply / Rollback): `secretsmanager:PutSecretValue`.

## Source of truth
The shipped code is embedded (base64) inside `secret-key-rotation.sh`. The
`skr-app/` folder is regenerated on each run and must not be treated as source.
