# Product — Secret Key Rotation Tool

A local-only web tool to safely add / update keys across many AWS Secrets
Manager secrets at once, then verify nothing was lost, with local backups and a
rollback path.

## Why it exists
Older ad-hoc scripts corrupted secret values that contain line breaks / escapes
(e.g. a PEM private key: `"-----BEGIN...\n...\n-----END...\n"`) because they
built JSON with shell strings and `jq`. This tool does all parsing, counting,
merging and verification locally with proper JSON engines and only touches AWS
to read the current value and — after explicit confirmation — to write the
final merged value.

## Operating principles
- Local only (`127.0.0.1`); never exposed to the internet.
- Correctness and visibility over speed: every add/replace and the key-count
  math is shown before any write.
- The operator must see and confirm the exact account (number + ARN) before a
  write happens.
- One secret at a time; a local backup before every write; rollback by path.

## Who uses it
An operator with AWS credentials on their machine, running one batch of key
insertions across ~dozens of Secrets Manager paths without disturbing existing
keys.
