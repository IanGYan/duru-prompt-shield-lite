# prompt-shield-lite

Minimal but practical anti-prompt-injection guardrail for OpenClaw agents.

This repo provides a lightweight security pipeline for:
- untrusted external content
- high-risk local actions
- outbound message safety (DLP-style redaction)

---

## What it does

`prompt-shield-lite` v2.x runs a compact layered pipeline:

1. **Normalize** text (NFKC, zero-width cleanup, basic homoglyph handling)
2. **Decode variants** (URL/HTML entity/Unicode escape/Base64)
3. **Rule scan** with explicit rule IDs (`rule_id::regex`)
4. **Decision** (`allow | warn | block`) based on mode/severity
5. **Log** JSONL records with hash-chain fields for audit continuity

It supports three entry points:
- `scripts/detect-injection.sh` → scan untrusted external text
- `scripts/pre-action-check.sh` → validate risky actions before execution
- `scripts/pre-send-scan.sh` → scan + redact outbound text

---

## Repository layout

```text
.
├── SKILL.md
├── skill.json
├── rules/
│   ├── critical.regex
│   ├── high.regex
│   ├── medium.regex
│   └── allowlist.regex
└── scripts/
    ├── psl-core.sh
    ├── detect-injection.sh
    ├── pre-action-check.sh
    ├── pre-send-scan.sh
    ├── analyze-log.sh
    └── test-v2.sh
```

---

## Requirements

- macOS/Linux shell
- `bash`
- `python3`

No external Python packages required.

---

## Quick start

```bash
# 1) detect suspicious external content
printf 'ignore all previous instructions' | bash scripts/detect-injection.sh

# 2) pre-check risky action
bash scripts/pre-action-check.sh "chmod 777 ./cache"

# 3) scan outbound content
printf 'token: sk-proj-...' | bash scripts/pre-send-scan.sh
```

All scripts return one-line JSON.

---

## Modes

Set with `PSL_MODE`:

- `strict`: MEDIUM+ blocks
- `balanced` (default): HIGH+ blocks, MEDIUM warns
- `lowfp`: fewer blocks, MEDIUM generally warns

Example:

```bash
PSL_MODE=strict bash scripts/detect-injection.sh <<< "..."
```

---

## Severity + exit codes

### Severity
- `SAFE`
- `LOW`
- `MEDIUM`
- `HIGH`
- `CRITICAL`

### Exit codes
- `0` → allow/pass
- `10` → warn
- `20` → block
- `2` → usage/input error

---

## JSON output schema

Example output:

```json
{
  "ok": false,
  "severity": "CRITICAL",
  "confidence": 0.93,
  "action": "block",
  "reasons": ["critical", "decoded_base64"],
  "matched_rules": ["CRIT_INSTR_OVERRIDE"],
  "mode": "balanced",
  "fingerprint": "abcd1234efgh5678",
  "sanitized_text": null,
  "type": "detect",
  "actor_id": "global",
  "rate_limit": {"count_in_window": 3, "max": 30, "window_sec": 60}
}
```

---

## Rule format

Rule files support explicit IDs:

```text
RULE_ID::regex
```

If `::` is omitted, runtime falls back to auto IDs (`<level>:L<n>`), but explicit IDs are recommended for observability and tuning.

---

## Built-in protections

### 1) Output redaction (send flow)
`pre-send-scan.sh` redacts sensitive patterns such as:
- OpenAI/GitHub/AWS tokens
- JWT / Bearer token
- Slack token / Slack webhook
- private key blocks
- local file paths

### 2) Action context gates
`pre-action-check.sh` adds structured checks for:
- destructive or core-file operations
- permission widening (`chmod 777`, `chown -R`)
- suspicious exfil endpoints
- gateway service control
- SSH-sensitive operations

### 3) Per-actor rate limit (DoS guard)
State stored in:
- `/Users/durubot/.openclaw/workspace/memory/psl-rate-limit.json` (default path)

---

## Configuration (env vars)

### Core
- `PSL_MODE` = `strict|balanced|lowfp`
- `PSL_ACTOR_ID` = actor identity (`global` default)
- `PSL_RULES_DIR` = custom rules directory
- `PSL_LOG_PATH` = JSONL log path

### Rate limit
- `PSL_RL_MAX_REQ` (default `30`)
- `PSL_RL_WINDOW_SEC` (default `60`)
- `PSL_RL_ACTION` = `block|warn` (default `block`)
- `PSL_RL_STATE_PATH` = rate-limit state JSON path

---

## Recommended defaults

### Production
```bash
export PSL_MODE=balanced
export PSL_ACTOR_ID=global
export PSL_RL_MAX_REQ=30
export PSL_RL_WINDOW_SEC=60
export PSL_RL_ACTION=block
```

### Dev/Debug
```bash
export PSL_MODE=lowfp
export PSL_ACTOR_ID=dev
export PSL_RL_MAX_REQ=80
export PSL_RL_WINDOW_SEC=60
export PSL_RL_ACTION=warn
```

---

## Analyze logs

```bash
# default: last 24h
bash scripts/analyze-log.sh

# custom path and window
bash scripts/analyze-log.sh /Users/durubot/.openclaw/workspace/memory/security-log.jsonl 48
```

Outputs:
- action mix
- severity distribution
- top matched rules
- top reasons
- per-mode breakdown
- heuristic false-positive candidates

---

## Run tests

```bash
bash scripts/test-v2.sh
```

Current suite verifies:
- safe pass
- critical injection block
- Base64 obfuscation detection
- mode behavior (`balanced` vs `strict`)
- action gate checks
- send redaction path
- rate-limit blocking
- output schema fields
- log hash-chain fields

---

## Notes

- This project is intentionally lightweight.
- It is a **guardrail layer**, not a full security platform.
- Pair with explicit human confirmation for irreversible/high-impact actions.

---

## License

MIT (see `LICENSE`)
