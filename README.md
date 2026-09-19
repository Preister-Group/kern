# KERN.open: The AI-First Security Orchestration CLI

> **One command. Three engines. Zero configuration.**
> Secrets, SAST and SCA  unified, deduplicated and AI-ready.

[![Version](https://img.shields.io/badge/version-1.0.0-blue)](package.json)
[![License](https://img.shields.io/badge/license-Apache--2.0-green)](LICENSE)
[![Node](https://img.shields.io/badge/node-%3E%3D16-brightgreen)](package.json)
[![Author](https://img.shields.io/badge/by-Preister%20Group-blueviolet)](#)

```bash
npm install -g kern.open
kern audit .
```

KERN wraps **Gitleaks**, **Horusec**, and **Trivy** into a single parallel orchestrator. It auto-downloads all engine binaries, deduplicates findings across engines via the Fusion Engine, and emits either a colour-coded human report or a clean, normalised JSON/SARIF payload,  ready for AI agents, CI pipelines, and GitHub Code Scanning.

---

## Table of Contents

1. [AI Agent Quick-Start](#ai-agent-quick-start)
2. [Installation](#installation)
3. [Commands](#commands)
4. [Audit Flags](#audit-flags)
5. [Output Modes](#output-modes)
6. [JSON Output Schema](#json-output-schema)
7. [SARIF Output](#sarif-output)
8. [--diff Mode](#--diff-mode)
9. [.kernignore](#kernignore)
10. [CI/CD Integration](#cicd-integration)
11. [Pre-commit Hook](#pre-commit-hook)
12. [kern doctor](#kern-doctor)
13. [Engines](#engines)
14. [Exit Codes](#exit-codes)
15. [Confidence & Severity Reference](#confidence--severity-reference)
16. [Repository Structure](#repository-structure)

---

## AI Agent Quick-Start

**This section is for AI coding agents (Cursor, Windsurf, Copilot, Claude, GPT, etc.).**
KERN is designed to be consumed programmatically. Follow this golden loop on every code change.

### The Golden Loop

```
kern doctor          -> run once on a new machine
kern setup           -> pre-download binaries (eliminates first-run latency)
kern audit . --json  -> scan; parse result.vulnerable
fix top issue        -> use result.issues[0] (sorted by confidence + severity)
kern audit . --json  -> re-scan; confirm fix; repeat
```

### Minimal Agent Integration (JavaScript)

```js
const { execSync } = require("child_process");

function kernAudit(path = ".") {
  const raw = execSync(`kern audit ${path} --json --silent`, {
    encoding: "utf8",
    stdio: ["pipe", "pipe", "pipe"],
  });
  return JSON.parse(raw);
}

const result = kernAudit(".");

if (!result.vulnerable) {
  console.log(" Clean! no findings over here");
  process.exit(0);
}

// Filter out engine noise
const real = result.issues.filter(
  (i) => i.type !== "SYSTEM_ERROR" && i.type !== "CONFIG_ERROR",
);

// Sort: confidence desc - > severity desc -> file > line
const CONF = { HIGH: 3, MEDIUM: 2, LOW: 1 };
const SEV = { CRITICAL: 4, HIGH: 3, MEDIUM: 2, LOW: 1, INFO: 0 };
real.sort((a, b) => {
  const cr = (CONF[b.confidence] ?? 0) - (CONF[a.confidence] ?? 0);
  if (cr !== 0) return cr;
  const sr = (SEV[b.severity] ?? 0) - (SEV[a.severity] ?? 0);
  if (sr !== 0) return sr;
  if (a.file !== b.file) return a.file < b.file ? -1 : 1;
  return (a.line || 0) - (b.line || 0);
});

// Act on the top finding
const top = real[0];
console.log(
  `[${top.severity}][${top.confidence}] ${top.file}:${top.line} — ${top.description}`,
);
if (top.suggested_fix) console.log(`Fix: ${top.suggested_fix}`);
```

### Fast Diff Mode for AI Agents (< 500 ms)

```bash
# Only scan git-modified files - ideal for agent inner loops
kern audit . --diff --json --silent
```

Engines that don't support single-file scanning (`supports_file_scan: false`) are automatically skipped in diff mode.

---

## Installation

```bash
# Global install (recommended)
npm install -g kern.open

# Verify
kern --version

# Pre-download all engine binaries (optional but recommended)
kern setup

# Health check
kern doctor
```

**No manual binary installation required.** KERN auto-downloads Gitleaks, Horusec, and Trivy on first use and caches them in `~/.kern/bin/`.

---

## Commands

| Command                        | Description                         |
| ------------------------------ | ----------------------------------- |
| `kern audit <path> [flags]`    | Scan a file or directory            |
| `kern doctor`                  | Environment health check            |
| `kern setup`                   | Pre-download all engine binaries    |
| `kern setup --force`           | Force re-download (wipes cache)     |
| `kern list`                    | List all registered engines         |
| `kern ignore list`             | Show current .kernignore exclusions |
| `kern ignore add <pattern>`    | Add a false-positive exclusion      |
| `kern ignore remove <pattern>` | Remove an exclusion                 |
| `kern --version` / `-v`        | Print version                       |
| `kern --help` / `-h`           | Print help                          |

---

## Audit Flags

```bash
kern audit <path> [flags]
```

| Flag              | Short | Description                                                |
| ----------------- | ----- | ---------------------------------------------------------- |
| `--engine <name>` | `-e`  | Run only the named engine (`gitleaks`, `horusec`, `trivy`) |
| `--silent`        | `-s`  | Suppress all progress output                               |
| `--json`          | `-j`  | Output raw JSON to stdout (machine-readable)               |
| `--format sarif`  |       | Output SARIF v2.1.0 to stdout (GitHub Code Scanning)       |
| `--diff`          |       | Only scan git-modified/staged/untracked files (fast mode)  |

### Examples

```bash
# Full project scan — human-readable
kern audit .

# Full project scan — JSON (AI agent / CI)
kern audit . --json

# Full project scan — silent JSON (clean stdout)
kern audit . --json --silent

# SARIF output for GitHub Code Scanning
kern audit . --format sarif > results.sarif

# Secrets only
kern audit . --json --engine gitleaks

# Dependency vulnerabilities only
kern audit . --json --engine trivy

# SAST only
kern audit . --json --engine horusec

# Fast diff scan (git-modified files only)
kern audit . --diff --json --silent

# Scan a specific file
kern audit path/to/file --json
```

---

## Output Modes

### Mode A — Human-readable (default)

```bash
kern audit .
```

**Colour-coded** terminal output with severity-grouped findings, file locations, descriptions, and suggested fixes. Use this when presenting results to a human developer.

### Mode B — JSON

```bash
kern audit . --json
```

Returns the **full normalised JSON** object (see [JSON Output Schema](#json-output-schema)). Use this when an AI agent or script is the consumer.

### Mode C — Silent JSON

```bash
kern audit . --json --silent
```

Identical to Mode B but suppresses all progress and download messages. Stdout contains only the JSON payload. Use in CI pipelines.

### Mode D — SARIF v2.1.0

```bash
kern audit . --format sarif
```

Emits a valid SARIF v2.1.0 document to stdout. All progress logs go to stderr. Use for GitHub Code Scanning, Azure DevOps, or any SARIF-compatible platform.

> **Clean stdout rule:** In `--json`, `--format sarif`, and `--silent` modes, **all** progress, warning, and informational messages are written to stderr. Stdout contains only the payload.

---

## JSON Output Schema

```jsonc
{
  "filename": "string", // scanned path (basename or absolute)
  "timestamp": "string", // ISO-8601 scan time
  "vulnerable": true, // true when real issues exist (type ≠ SYSTEM_ERROR)
  "issues": [
    /* Issue[] — see below */
  ],
}
```

### Issue Object

```jsonc
{
  "engines": ["gitleaks", "horusec"], // all engines that detected this finding
  "confidence": "MEDIUM", // HIGH (3+ engines) | MEDIUM (2) | LOW (1)
  "type": "SECRET", // finding category
  "severity": "CRITICAL", // CRITICAL | HIGH | MEDIUM | LOW | INFO
  "file": "src/config.js", // relative path to affected file
  "line": 42, // lowest line number across engines
  "description": "Gitleaks: GitHub Token; Horusec: Hardcoded credential",
  "suggested_fix": "Move to environment variable or secrets manager",
}
```


### Special Issue Types

| Type           | Meaning                           | Sets `vulnerable`? | Triggers exit 1? |
| -------------- | --------------------------------- | ------------------ | ---------------- |
| `SYSTEM_ERROR` | Engine failed to run              | No                 | No               |
| `CONFIG_ERROR` | Engine name not found in manifest | No                 | No               |

When `SYSTEM_ERROR` appears, run `kern doctor` to diagnose.

### Full Example

```json
{
  "filename": "project",
  "timestamp": "2026-03-24T10:00:00.000Z",
  "vulnerable": true,
  "issues": [
    {
      "engines": ["gitleaks", "horusec"],
      "confidence": "MEDIUM",
      "type": "SECRET",
      "severity": "CRITICAL",
      "file": "src/db.js",
      "line": 7,
      "description": "Gitleaks: AWS Access Key; Horusec: Hardcoded credential",
      "suggested_fix": "Move to process.env.AWS_ACCESS_KEY_ID"
    },
    {
      "engines": ["trivy"],
      "confidence": "LOW",
      "type": "VULN",
      "severity": "HIGH",
      "file": "package.json",
      "line": 12,
      "description": "CVE-2023-XXXX: lodash < 4.17.21 — prototype pollution"
    }
  ]
}
```

---

## SARIF Output

SARIF (Static Analysis Results Interchange Format) v2.1.0 is the standard format for uploading security findings to GitHub Code Scanning, Azure DevOps, and other platforms.

```bash
kern audit . --format sarif > results.sarif
```

### SARIF Document Structure

```jsonc
{
  "$schema": "https://schemastore.azurewebsites.net/schemas/json/sarif-2.1.0.json",
  "version": "2.1.0",
  "runs": [
    {
      "tool": {
        "driver": {
          "name": "KERN",
          "version": "1.0.0",
          "rules": [
            /* one rule per unique finding type */
          ],
        },
      },
      "results": [
        {
          "ruleId": "SECRET",
          "level": "error", // CRITICAL/HIGH → error | MEDIUM → warning | LOW → note
          "message": { "text": "AWS Access Key detected" },
          "locations": [
            {
              "physicalLocation": {
                "artifactLocation": {
                  "uri": "src/db.js",
                  "uriBaseId": "%SRCROOT%",
                },
                "region": { "startLine": 7 },
              },
            },
          ],
          "properties": { "tags": ["gitleaks", "horusec"] },
        },
      ],
    },
  ],
}
```

### Severity → SARIF Level Mapping

| KERN Severity      | SARIF Level |
| ------------------ | ----------- |
| `CRITICAL`, `HIGH` | `error`     |
| `MEDIUM`           | `warning`   |
| `LOW`, `INFO`      | `note`      |

### GitHub Code Scanning Upload

```yaml
- name: KERN Security Scan (SARIF)
  run: kern audit . --format sarif > kern-results.sarif

- name: Upload SARIF to GitHub
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: kern-results.sarif
```

---

## --diff Mode

`--diff` mode makes KERN scan only the files that have changed in the current git working tree — unstaged changes, staged changes, and untracked new files.

```bash
kern audit . --diff --json --silent
```

### How It Works

1. Runs `git diff --name-only` (unstaged)
2. Runs `git diff --cached --name-only` (staged)
3. Runs `git ls-files --others --exclude-standard` (untracked)
4. Deduplicates and filters to files that exist on disk
5. Passes the file list to engines that support `supports_file_scan: true`
6. Engines with `supports_file_scan: false` (e.g. Horusec) are **skipped** automatically

### Use Cases

| Scenario                | Command                               |
| ----------------------- | ------------------------------------- |
| AI agent inner loop     | `kern audit . --diff --json --silent` |
| Pre-commit hook (fast)  | `kern audit . --diff --json`          |
| PR review (staged only) | `kern audit . --diff --json`          |

### Requirements

- Must be inside a git repository
- `git` must be installed and on PATH (verified by `kern doctor`)
- At least one modified/staged/untracked file must exist

If not in a git repo, KERN falls back to a full scan with a warning.

---

## .kernignore

KERN reads `.kernignore` in the project root to exclude paths from scanning. It is auto-created with sensible defaults on first run.

### Default .kernignore

```
# KERN Default Ignores
node_modules
.git
.kern_bin
.horusec
dist
build
coverage
.next
.vscode
.idea
```

### Managing Exclusions

```bash
kern ignore list                    # show current exclusions
kern ignore add tests/fixtures/     # suppress a false positive path
kern ignore add "**/*.test.js"      # suppress test files
kern ignore remove tests/fixtures/  # re-enable a path
```

**Rules:**

- Never add broad patterns like `src` or `.`
- Always document why a pattern was suppressed (in commit message or PR)
- After adding an entry, re-run `kern audit . --json` to confirm the finding is gone
- Never delete `.kernignore`

---

## CI/CD Integration

### GitHub Actions — Full Scan

```yaml
name: KERN Security Audit

on: [push, pull_request]

jobs:
  kern-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install KERN
        run: npm install -g kern.open

      - name: Pre-download engine binaries
        run: kern setup

      - name: Run KERN audit
        run: |
          kern audit . --json --silent > kern-report.json
          node -e "
            const r = require('./kern-report.json');
            const critical = r.issues.filter(i =>
              ['CRITICAL','HIGH'].includes(i.severity) &&
              !['SYSTEM_ERROR','CONFIG_ERROR'].includes(i.type)
            );
            if (critical.length > 0) {
              console.error('KERN: ' + critical.length + ' critical/high issue(s) found.');
              critical.forEach(i =>
                console.error('[' + i.severity + '][' + i.confidence + '] ' +
                  i.file + ':' + i.line + ' — ' + i.description)
              );
              process.exit(1);
            }
            console.log('KERN: Audit passed — ' + r.issues.length + ' total findings (none critical/high).');
          "
```

### GitHub Actions - PR Diff Scan (Fast)

```yaml
name: KERN PR Diff Scan

on: [pull_request]

jobs:
  kern-diff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0 # required for git diff

      - name: Install KERN
        run: npm install -g kern.open && kern setup

      - name: KERN diff scan
        run: kern audit . --diff --json --silent > kern-diff.json

      - name: Check results
        run: |
          node -e "
            const r = require('./kern-diff.json');
            if (r.vulnerable) {
              r.issues
                .filter(i => !['SYSTEM_ERROR','CONFIG_ERROR'].includes(i.type))
                .forEach(i => console.error('[' + i.severity + '] ' + i.file + ':' + i.line + ' — ' + i.description));
              process.exit(1);
            }
            console.log('KERN diff: clean.');
          "
```

### GitHub Actions - SARIF Upload to Code Scanning

```yaml
name: KERN SARIF Upload

on: [push]

jobs:
  kern-sarif:
    runs-on: ubuntu-latest
    permissions:
      security-events: write

    steps:
      - uses: actions/checkout@v4

      - name: Install KERN
        run: npm install -g kern.open && kern setup

      - name: Run KERN (SARIF)
        run: kern audit . --format sarif > kern-results.sarif

      - name: Upload SARIF to GitHub Code Scanning
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: kern-results.sarif
```

### Dockerfile Pre-bake

```dockerfile
# Eliminate first-run binary download latency
RUN npm install -g kern.open && kern setup
```

---

## Pre-commit Hook

Save as `.git/hooks/pre-commit` and run `chmod +x .git/hooks/pre-commit`:

```sh
#!/bin/sh
echo " Running KERN security audit..."

RESULT=$(kern audit . --json --silent 2>/dev/null)
EXIT=$?

if [ $EXIT -eq 1 ]; then
  echo " KERN: Security issues found. Commit blocked."
  echo "$RESULT" | node -e \
    "process.stdin.resume();let d='';process.stdin.on('data',c=>d+=c);\
     process.stdin.on('end',()=>{const r=JSON.parse(d);\
     r.issues.filter(i=>['CRITICAL','HIGH'].includes(i.severity) && \
     !['SYSTEM_ERROR','CONFIG_ERROR'].includes(i.type))\
     .forEach(i=>console.log('['+i.severity+']['+i.confidence+'] '+i.file+':'+i.line+' — '+i.description));});"
  exit 1
fi

if [ $EXIT -eq 2 ]; then
  echo "  KERN: Audit failed to run. For more info, run: kern doctor"
  exit 1
fi

echo "KERN: No critical/high issues found."
```

**Fast variant (diff only - recommended for LARGE repos):**

```sh
#!/bin/sh
RESULT=$(kern audit . --diff --json --silent 2>/dev/null)
[ $? -eq 1 ] && echo " KERN: Issues in modified files." && echo "$RESULT" && exit 1
echo " KERN diff: clean."
```

---

## kern doctor

Run `kern doctor` whenever the script behaves unexpectedly or before onboarding a new machine.


### When to Run

| Situation                      | Action                               |
| ------------------------------ | ------------------------------------ |
| Fresh install on new machine   | `kern doctor` before first audit     |
| Engine produces no output      | `kern doctor`                        |
| Binary download fails          | `kern doctor` → `kern setup --force` |
| SHA-256 mismatch               | `kern setup --force`                 |
| `SYSTEM_ERROR` in audit output | `kern doctor`                        |
| Network/firewall change        | `kern doctor`                        |
| CI job fails with binary error | `kern doctor`                        |

---

## Engines

| Engine       | Type            | `supports_file_scan`  | What it detects                                                     |
| ------------ | --------------- | --------------------- | ------------------------------------------------------------------- |
| **Gitleaks** | Secrets scanner | ✅ `true`             | API keys, tokens, passwords, private keys, connection strings       |
| **Horusec**  | SAST            | ❌ `false` (dir-only) | Hardcoded credentials, injection flaws, insecure patterns           |
| **Trivy**    | SCA             | ✅ `true`             | Vulnerable dependency versions (CVEs) in package.json, go.mod, etc. |

> `supports_file_scan: false` means the engine requires a directory target. In `--diff` mode, these engines are automatically skipped. Horusec's results are post-filtered by the parser to match the diff file list.

All binaries are auto-downloaded from HuggingFace on first use. No manual installation required.

---

## Exit Codes

| Code | Meaning                                                                 |
| ---- | ----------------------------------------------------------------------- |
| `0`  | Clean scan — no real vulnerabilities (engine warnings don't count)      |
| `1`  | Real vulnerabilities found (`type ≠ SYSTEM_ERROR` and `≠ CONFIG_ERROR`) |
| `2`  | KERN itself failed to run (system-level error — run `kern doctor`)      |

> **Important:** Do not treat exit `0` as "no vulnerabilities" without also checking `result.vulnerable` in JSON mode. Engine errors produce exit `0` but may indicate incomplete scans.

---

## Confidence & Severity Reference

### Confidence

| Value    | Meaning          | Action                                                |
| -------- | ---------------- | ----------------------------------------------------- |
| `HIGH`   | 3+ engines agree | Fix immediately — confirmed finding                   |
| `MEDIUM` | 2 engines agree  | Fix immediately — very likely real                    |
| `LOW`    | 1 engine only    | Investigate first; fix or suppress with justification |

### Severity

| Value      | Examples                                                   | Action                                               |
| ---------- | ---------------------------------------------------------- | ---------------------------------------------------- |
| `CRITICAL` | AWS keys, root credentials, active cloud secrets           | Rotate immediately. Assume already compromised.      |
| `HIGH`     | API tokens, JWT keys, SSH private keys, known-exploit CVEs | Fix before committing. Rotate or upgrade.            |
| `MEDIUM`   | Generic passwords, moderate CVEs, weak crypto config       | Fix within current sprint.                           |
| `LOW`      | Low-severity CVEs, potential test data                     | Review: fix, suppress, or document as accepted risk. |
| `INFO`     | Audit trail entries, informational notices                 | No action required unless policy demands it.         |

---

## Repository Structure

```
kern.open/
├── bin/
│   └── kern.js                  # CLI entry point — argument parsing, output formatting
├── lib/
│   ├── core.js                  # Audit orchestrator + Fusion Engine
│   ├── engines_manifest.js      # Single source of truth for all engines
│   └── engines/
│       ├── gitleaks.js          # Gitleaks parser
│       ├── horusec.js           # Horusec parser
│       └── trivy.js             # Trivy parser
│   └── utils/
│       ├── downloader.js        # Binary download + integrity verification
│       ├── doctor.js            # Environment health checker
│       └── setup.js             # Binary pre-download utility
├── .cursorrules                 # AI agent integration rules (Cursor)
├── .windsurfrules               # AI agent integration rules (Windsurf)
├── .kernignore                  # Scan exclusion patterns (auto-created)
├── package.json
└── README.md
```

---

_Developed by **Preister Group** — KERN v1.0.0_
_Contact BobPotat0 if you have any issues_
