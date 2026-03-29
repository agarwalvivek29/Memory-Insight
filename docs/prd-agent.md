# PRD: Go Agent Daemon

**Service:** `agent/`
**Language:** Go 1.22
**Distribution:** Single static binary via GitHub Releases
**Runtime:** systemd service on Linux (amd64 / arm64), runs as root

---

## Overview

The Go agent is the only piece of software the user installs on their servers. It must be a single static binary — no runtime dependencies, no Docker, no package manager. It registers the machine with the API, long-polls for jobs, takes memory dumps via AVML, uploads them directly to S3, and reports success or failure back to the API.

The agent never listens on any port. All communication is outbound-only.

---

## Goals

- Single static binary (~50MB with embedded AVML)
- Curl-installable in one command
- No inbound ports — works through NAT, firewalls, cloud security groups
- Org API key used once at registration, then discarded (machine token stored instead)
- Survives transient network failures with exponential backoff
- Clean failure reporting: every error path calls POST /jobs/:id/fail with a structured error code

## Non-goals

- Windows / macOS support (AVML is Linux-only)
- Alpine Linux (AVML does not support musl libc)
- Running without root (AVML requires CAP_SYS_RAWIO)

---

## Module structure

```
agent/
├── cmd/
│   └── meminsight/
│       └── main.go          Entry point — register or daemon subcommand
├── internal/
│   ├── config/
│   │   └── config.go        Read/write /etc/meminsight/config.yaml
│   ├── register/
│   │   └── register.go      POST /machines/register, receive machine token
│   ├── poller/
│   │   └── poller.go        Long-poll loop: GET /jobs/pending with reconnect logic
│   ├── dumper/
│   │   └── dumper.go        Preflight disk check + AVML invocation
│   ├── uploader/
│   │   └── uploader.go      Presigned URL fetch + S3 multipart (64MB chunks, 3-retry)
│   └── reporter/
│       └── reporter.go      POST /jobs/:id/complete and /fail
├── avml/
│   ├── avml-linux-amd64     Embedded AVML binary (~25MB)
│   └── avml-linux-arm64     Embedded AVML binary (~25MB)
├── install.sh               Install script (downloaded by curl one-liner)
├── meminsight-agent.service systemd unit file
├── go.mod
└── Makefile
```

---

## Config file

Written to `/etc/meminsight/config.yaml` after successful registration.

```yaml
machine_id:    mch_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
machine_token: mt_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
api_url:       https://api.memoryinsight.io
```

**The org API key (`sk_live_xxxx`) is never written to this file.**

---

## Install flow

```bash
curl -sSL https://get.memoryinsight.io | bash -s -- \
  --key sk_live_xxxx \
  --namespace ns_xxxxxxxx
```

The install script (`install.sh`):

1. Detect arch: `uname -m` → `amd64` or `arm64`
2. Download `meminsight-agent-linux-{arch}` from GitHub Releases latest tag
3. Copy to `/usr/local/bin/meminsight-agent`, `chmod +x`
4. Run `meminsight-agent register --key sk_live_xxxx --namespace ns_xxxxxxxx`
5. Copy `meminsight-agent.service` to `/etc/systemd/system/`
6. `systemctl daemon-reload && systemctl enable --now meminsight-agent`

Errors at any step are printed and exit 1. No silent failures.

---

## Registration

```
meminsight-agent register --key sk_live_xxxx --namespace ns_xxxxxxxx
```

1. Detect hostname (`os.Hostname()`), OS info (`/etc/os-release`), arch, RAM (`/proc/meminfo`)
2. POST `/machines/register` with `Authorization: Bearer sk_live_xxxx`
3. On 201: write `config.yaml` with `machine_id` + `machine_token`. Org key is NOT written.
4. On 409: print "machine already registered for this hostname in namespace. To re-register, deregister first from the UI." Exit 1.
5. On 401/403: print error message from API response. Exit 1.
6. On network error: retry 3× with 5s backoff, then exit 1.

---

## Daemon loop

```
systemd starts: meminsight-agent daemon
  → reads /etc/meminsight/config.yaml
  → starts poller goroutine
  → starts heartbeat goroutine (PATCH /machines/:id/heartbeat every 60s)

Poller loop (goroutine):
  for {
    job, err := poller.Poll(ctx)   // GET /jobs/pending, hangs 30s
    if err != nil {
      // network error: sleep 5s, retry (exponential backoff up to 5min)
      continue
    }
    if job == nil {
      // 204 timeout: re-poll immediately
      continue
    }
    // job received: run dump pipeline in goroutine
    go runJob(ctx, job)
  }
```

Only one job runs at a time per agent. If a job is already running when a poll returns a new job, the new job is acknowledged but queued locally (FIFO, max depth 1 — additional jobs are rejected with a 409-style response to API).

---

## Dump pipeline

```
runJob(ctx, job):
  1. Pre-flight: df /tmp → need ≥ 2× total_memory_gb free. Fail: DISK_FULL
  2. Pre-flight: os.Getuid() == 0. Fail: PERMISSION_DENIED
  3. Extract AVML from embedded bytes:
       data, _ := avmlBinaries.ReadFile("avml/avml-linux-" + runtime.GOARCH)
       write to /tmp/avml-{version}, chmod 0755
  4. Run: exec.Command("/tmp/avml-{version}", dumpPath).Run()
       Fail (non-zero exit): AVML_FAILED, cleanup
  5. GET /dumps/presigned-url?job_id=&size_bytes=
  6. Multipart PUT to S3:
       chunk size: 64MB
       retry per chunk: 3× exponential backoff (5s, 25s, 125s)
       on exhaustion: UPLOAD_FAILED
  7. SHA-256 checksum of dump file
  8. POST /jobs/:id/complete { s3_key, size_bytes, checksum }
  9. rm /tmp/meminsight-{job_id}.lime
 10. rm /tmp/avml-{version}   (if no other job running)
```

---

## AVML embedding

```go
//go:embed avml/avml-linux-amd64 avml/avml-linux-arm64
var avmlBinaries embed.FS
```

AVML version is pinned in `go.mod`-adjacent `avml/VERSION` file. Upgrade by replacing the binaries and bumping `VERSION`. The extracted binary path is `/tmp/avml-{VERSION}` — if it already exists with the correct hash, skip extraction.

---

## Error codes

| Code | Condition | Retry |
|------|-----------|-------|
| `AVML_FAILED` | AVML non-zero exit | No |
| `UPLOAD_FAILED` | S3 multipart exhausted after 3 retries | No (already retried) |
| `DISK_FULL` | /tmp has < 2× RAM free | No |
| `PERMISSION_DENIED` | Not running as root | No |

---

## systemd unit

```ini
[Unit]
Description=Memory Insight Agent
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/local/bin/meminsight-agent daemon
Restart=on-failure
RestartSec=10s
User=root

[Install]
WantedBy=multi-user.target
```

---

## GitHub Actions release

Trigger: `push` to tag matching `v*`

Steps:
1. `go build -ldflags="-s -w" -o meminsight-agent-linux-amd64 ./cmd/meminsight` (GOOS=linux GOARCH=amd64)
2. Same for arm64
3. `gh release create $TAG meminsight-agent-linux-amd64 meminsight-agent-linux-arm64`
4. Update `install.sh` version pin (or use `latest` release API in script)

Cross-compilation from GitHub-hosted runner (ubuntu-latest) — no Docker needed.

---

## Acceptance criteria (Phase 1)

- [ ] `curl ... | bash` succeeds on Ubuntu 22.04, Debian 12, Amazon Linux 2023, RHEL 9
- [ ] `config.yaml` contains `machine_id` and `machine_token`. No `api_key` or `org_key` field present.
- [ ] Duplicate registration (same hostname + namespace) prints clear error, exits 1, no partial state
- [ ] Daemon survives API restart: re-polls successfully within 1 poll cycle
- [ ] Daemon survives Redis restart: long-poll returns 204 (not panic), re-polls immediately
- [ ] DISK_FULL preflight fires before AVML starts (no partial dump left behind)
- [ ] PERMISSION_DENIED: job fails immediately with correct error code, not a confusing AVML error
- [ ] UPLOAD_FAILED: retried 3× before calling /fail, dump file cleaned up regardless
- [ ] Binary is statically linked (`file meminsight-agent-linux-amd64` shows "statically linked")
- [ ] Binary size < 60MB (50MB AVML + 10MB Go overhead)
