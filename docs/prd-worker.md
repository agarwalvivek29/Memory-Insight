# PRD: Volatility3 Analysis Worker

**Service:** `worker/`
**Language:** Python 3.11
**Runtime:** AWS ECS Fargate
**Trigger:** ECS RunTask from Lambda dispatcher
**Images:** `ghcr.io/memoryinsight/worker` (dev) + ECR (Fargate)

---

## Overview

The worker is a short-lived Fargate task that runs once per memory dump. It downloads the dump from S3, detects the kernel version, runs Volatility3 plugins in a network-isolated subprocess, classifies findings by severity, generates Claude AI summaries for the top 10 findings, and writes everything to Postgres.

The worker is the most security-sensitive component — it processes potentially hostile memory dumps from compromised machines. Network isolation is non-negotiable.

---

## Goals

- Run Volatility3 plugins against a memory dump and produce structured findings
- Classify findings by severity using the rule table from `ARCHITECTURE.md`
- Generate plain-English AI summaries for the top 10 findings by severity
- Write all results to Postgres and mark the analysis complete
- Complete within the per-size SLA (5 min small, 15 min medium, 40 min large)

## Non-goals

- Diff alerting (Phase 2)
- ML severity scoring (Phase 3)
- Threat intel enrichment (Phase 3)
- Supporting Alpine Linux / musl libc kernels

---

## Network isolation model

```
ECS Fargate task
├── Parent Python process
│   ├── Network: YES (HTTPS only via NAT gateway)
│   │   - S3 download (via VPC endpoint, no internet)
│   │   - Postgres writes
│   │   - Claude API calls
│   │   - SSM parameter fetch (startup only)
│   └── Spawns subprocess for each Volatility3 plugin:
│
└── Volatility3 subprocess (per plugin)
    ├── Network: NONE (unshare --net)
    └── Reads dump from /tmp, writes JSON to stdout
```

`unshare --net` creates a new network namespace with no interfaces. Even if Volatility3 or a plugin had a supply chain compromise, it cannot exfiltrate data.

---

## Module structure

```
worker/
├── main.py                  Entry point: reads JOB_ID, S3_KEY, ANALYSIS_ID from env
├── plugins/
│   ├── runner.py            Volatility3 subprocess runner (unshare --net)
│   ├── pslist.py            Process list plugin wrapper
│   ├── netscan.py           Network connections plugin wrapper
│   ├── malfind.py           Malicious memory region detection wrapper
│   ├── cmdline.py           Process command line arguments wrapper
│   ├── proc_maps.py         Linux process memory maps wrapper
│   └── envars.py            Process environment variables wrapper
├── classify.py              Rule-based severity classification
├── ai/
│   └── summarizer.py        Claude API integration (top 10 findings)
├── db.py                    Postgres writes (psycopg2)
├── s3.py                    Dump download helper
├── ssm.py                   Claude API key fetch from SSM at startup
├── isf/                     Bundled Volatility3 ISF symbol files
│   ├── ubuntu-22.04/
│   ├── ubuntu-20.04/
│   ├── debian-12/
│   ├── amzn-2023/
│   └── rhel-9/
├── Dockerfile
└── requirements.txt
```

---

## Environment variables (set by ECS RunTask)

```
JOB_ID          UUID of the job
S3_KEY          S3 object key for the dump file
ANALYSIS_ID     UUID of the pre-created analysis record
DATABASE_URL    Postgres connection string
AWS_REGION      us-east-1
S3_BUCKET       memoryinsight-dumps
SSM_CLAUDE_KEY  /memoryinsight/claude-api-key (SSM parameter path)
```

---

## Execution flow

```python
# main.py
1. Fetch Claude API key from SSM (boto3, at startup, fail fast if unavailable)
2. Update analysis record: status = 'running', started_at = now()
3. Download dump: s3.download(S3_KEY) → /tmp/dump.lime
4. Detect kernel: strings /tmp/dump.lime | grep "Linux version" | head -1
5. Match kernel to bundled ISF directory
   → no match: update analysis status = 'failed', error = 'UNSUPPORTED_KERNEL', exit 0
6. For each plugin [malfind, pslist, netscan, cmdline, proc_maps, envars]:
   raw_output = runner.run(plugin, '/tmp/dump.lime', isf_path)
   findings = plugin_module.parse(raw_output)
   classified = classify.apply(findings, plugin)
   db.write_findings(ANALYSIS_ID, classified)
7. Top 10 findings by severity → Claude API → ai_summary per finding
8. Update analysis record: status = 'done', completed_at = now(), plugins_run = [...]
9. Update job record: status = 'DONE', completed_at = now()
10. Delete /tmp/dump.lime
```

---

## Plugin runner

```python
# runner.py
import subprocess, json, shlex

def run(plugin_name: str, dump_path: str, isf_path: str) -> dict:
    cmd = [
        'unshare', '--net',
        'python3', '-m', 'volatility3',
        '-f', dump_path,
        '--symbol-dirs', isf_path,
        plugin_name,
        '--output', 'json'
    ]
    result = subprocess.run(
        cmd,
        capture_output=True,
        text=True,
        timeout=300  # 5 min per plugin max
    )
    if result.returncode != 0:
        raise PluginError(plugin_name, result.stderr)
    return json.loads(result.stdout)
```

---

## Severity classification rules

```python
# classify.py
RULES = {
    'malfind': [
        # (condition_fn, severity)
        (lambda r: r['protection'] == 'PAGE_EXECUTE_READWRITE' and not r.get('file_output'), 'CRITICAL'),
        (lambda r: 'X' in r.get('protection', ''), 'HIGH'),
        (lambda r: True, 'HIGH'),  # default for malfind
    ],
    'pslist': [
        (lambda r: r.get('hollow', False), 'CRITICAL'),
        (lambda r: r.get('pid') != r.get('tree_pid'), 'HIGH'),
        (lambda r: True, 'INFO'),
    ],
    'netscan': [
        (lambda r: r.get('foreign_addr') and _is_non_rfc1918(r['foreign_addr'])
                   and r.get('local_port') not in (80, 443, 22), 'HIGH'),
        (lambda r: True, 'MEDIUM'),
    ],
    'cmdline': [
        (lambda r: _has_base64_args(r.get('args', '')), 'HIGH'),
        (lambda r: _shell_from_non_shell_parent(r), 'HIGH'),
        (lambda r: True, 'LOW'),
    ],
    'linux.proc.Maps': [
        (lambda r: r.get('path', '').startswith(('/tmp/', '/home/')), 'CRITICAL'),
        (lambda r: not r.get('path') and 'x' in r.get('perms', ''), 'HIGH'),
        (lambda r: True, 'MEDIUM'),
    ],
    'envars': [
        (lambda r: any(k in r.get('variable', '').upper()
                       for k in ('KEY', 'SECRET', 'PASSWORD', 'TOKEN')), 'HIGH'),
        (lambda r: True, 'LOW'),
    ],
}
```

---

## AI summarizer

```python
# ai/summarizer.py
# Top 10 findings by severity (CRITICAL > HIGH > MEDIUM > LOW > INFO)
# ~150 token prompt + raw excerpt → ~100 token output
# Max cost: ~$0.015/job at claude-haiku pricing

SEVERITY_ORDER = ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW', 'INFO']

def summarise_top_findings(findings: list[dict], api_key: str) -> list[dict]:
    sorted_findings = sorted(findings, key=lambda f: SEVERITY_ORDER.index(f['severity']))
    top10 = sorted_findings[:10]

    client = anthropic.Anthropic(api_key=api_key)
    for finding in top10:
        prompt = (
            f"Memory forensics finding from plugin {finding['plugin']}.\n"
            f"Severity: {finding['severity']}\n"
            f"Raw data: {json.dumps(finding['raw_output'])[:500]}\n\n"
            "Explain in 2-3 sentences what this finding means for a sysadmin "
            "and whether they should be concerned. Plain English, no jargon."
        )
        response = client.messages.create(
            model='claude-haiku-4-5-20251001',
            max_tokens=150,
            messages=[{'role': 'user', 'content': prompt}]
        )
        finding['ai_summary'] = response.content[0].text

    return findings
```

---

## Kernel detection and ISF matching

```python
def detect_kernel(dump_path: str) -> str | None:
    result = subprocess.run(
        ['strings', dump_path],
        capture_output=True, text=True, timeout=60
    )
    for line in result.stdout.splitlines():
        m = re.search(r'Linux version (\S+)', line)
        if m:
            return m.group(1)
    return None

def find_isf(kernel_version: str, isf_root: str) -> str | None:
    # Match kernel version prefix to bundled ISF directories
    # e.g. "5.15.0-91-generic" → ubuntu-22.04/
    # Returns path to matching ISF dir or None
```

ISF files sourced from https://github.com/Abyss-W4tcher/volatility3-symbols.
Bundled at Docker build time. CDN pull is forbidden (violates post-startup isolation).

Supported kernels (Phase 1):
- Ubuntu 22.04 (5.15.x)
- Ubuntu 20.04 (5.4.x)
- Debian 12 (6.1.x)
- Amazon Linux 2023 (6.1.x)
- RHEL 9 / Rocky 9 (5.14.x)

---

## Docker image

```dockerfile
FROM python:3.11-slim

# System deps for Volatility3 and unshare
RUN apt-get update && apt-get install -y \
    git binutils libdwarf-dev libelf-dev \
    util-linux \    # provides unshare
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Bundle ISF files (baked at build time)
COPY isf/ /app/isf/

COPY . .
CMD ["python3", "main.py"]
```

Images pushed on merge to main:
- `ghcr.io/memoryinsight/worker:latest` — local dev
- `{account}.dkr.ecr.us-east-1.amazonaws.com/memoryinsight-worker:latest` — Fargate

---

## Fargate task definitions

Three task definitions, differing only in vCPU, memory, and ephemeralStorageGiB:

| Definition | vCPU | Memory | ephemeralStorageGiB | Dump size |
|-----------|------|--------|---------------------|-----------|
| worker-small | 2 | 8192 | 32 | < 4 GB |
| worker-medium | 4 | 16384 | 64 | 4–16 GB |
| worker-large | 8 | 65536 | 200 | > 16 GB |

`ephemeralStorageGiB` must be set explicitly — Fargate default (20GB) is insufficient for dumps >14GB.

---

## Acceptance criteria (Phase 1)

- [ ] Worker processes a real Ubuntu 22.04 dump end-to-end: downloads, detects kernel, runs all 6 plugins, writes findings, generates summaries, marks analysis done
- [ ] Volatility3 subprocess cannot reach the internet (`unshare --net` confirmed via test)
- [ ] Unsupported kernel version: analysis record shows `status = 'failed'`, `error = 'UNSUPPORTED_KERNEL'`
- [ ] malfind on a known RWX page produces CRITICAL finding with AI summary
- [ ] envars with PASSWORD in variable name produces HIGH finding
- [ ] Worker completes a 4GB dump in < 5 minutes on worker-small
- [ ] Claude API key fetched from SSM at startup, not from env var
- [ ] /tmp/dump.lime deleted after analysis (success or failure)
- [ ] Docker image pushed to both GHCR and ECR on merge to main
