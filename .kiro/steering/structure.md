# Structure — Secret Key Rotation Tool (portable)

## Layout
```
secret_key_rotation_portable/
├── secret-key-rotation.sh  # THE one runnable file (writes app, sets up, launches)
├── README.md               # usage + issues handled
├── CLAUDE.md / AGENTS.md    # agent context + rules
├── .kiro/steering/          # Kiro steering (product / tech / structure)
└── skr-app/                 # created on first run (git-ignored)
    ├── server.py            # backend (extracted from the .sh)
    ├── index.html           # UI (extracted from the .sh)
    ├── .venv/               # local virtualenv (boto3)
    ├── accounts.cache.json  # per-machine discovered-account cache
    └── backups/             # per-run local backups
        ├── <account>_<ts>/           # scan runs (rollback restore source)
        ├── <account>_<ts>_APPLY/     # pre-write snapshots
        └── <account>_<ts>_ROLLBACK/  # pre-rollback snapshots
```

## Where things go
- **The shipped source** is embedded (base64) inside `secret-key-rotation.sh`
  (both `server.py` and `index.html`). That is the source of truth.
- `skr-app/` is regenerated on every run — never treat it as source, and it is
  git-ignored along with its runtime data.
- **No secrets, accounts, or machine paths in the committed files.**

## Endpoint ↔ stage map
| UI section | Endpoint | Writes? |
|---|---|---|
| 1 Find accounts | `POST /api/discover-accounts`, `GET /api/saved-accounts` | no |
| 1 Check identity | `POST /api/check-identity` | no |
| 1 Check permissions | `POST /api/check-permissions` | no |
| 2 Scan & backup | `POST /api/scan-backup` | local backup only |
| 2 jq Verify | `POST /api/jq-verify` | no |
| 4 Preview | `POST /api/preview` | no |
| 4 Apply (confirm) | `POST /api/apply` | YES (gated) |
| 5 Rollback preview | `POST /api/rollback-preview` | no |
| 5 Rollback (confirm) | `POST /api/rollback-apply` | YES (gated) |

## Backup file shape
Each backup file is named by its secret path (`/` → `__`) and stores:
`secret_path`, `account`, `region`, `version_id`, `backed_up_at`, and the raw
`secret_string`. Scan runs also write `_manifest.json` with per-path
`keys_before` (used for the scan→apply drift check and rollback).
