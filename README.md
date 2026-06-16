# olefied

**A production-ready front-end for [Heinlein's olefy](https://github.com/HeinleinSupport/olefy).**

olefy is a small TCP service that lets [rspamd](https://rspamd.com) hand Office
attachments to [oletools](https://github.com/decalage2/oletools)' `olevba`, which
parses them for VBA macros, OLE tricks and other malicious-document tells — the
kind of thing that hides in a "weird invoice" `.docm`.

olefy does that job well, but it scans **one file at a time** and has **no scan
timeout**, so a single hostile document can stall the whole service. That's fine
on a quiet box and a problem on a busy mail relay.

**olefied wraps the upstream `olefy.py` unchanged and adds the things a real mail
stream needs:** concurrency, a scan timeout, backpressure, input limits, self-healing,
and an optional result cache. Point rspamd at olefied instead of olefy and nothing
else changes — same wire protocol, same `PONG`.

Everything olefied adds lives in one file, [`docker/olefyd.py`](docker/olefyd.py);
the upstream `olefy.py` is used **verbatim** as the worker.

📖 Full write-up — why Office macros still bite, where stock olefy falls over under
load, and how olefied fixes it:
**[Olefy and rspamd: scan Office macro malware in your mail](https://deb.myguard.nl/2026/06/olefy-rspamd-office-macro-scanning/)**

---

## What olefied adds

| | Stock olefy | olefied |
|---|---|---|
| **Concurrency** | one scan at a time | a pool of workers, load-balanced (`olevba` is CPU-bound) |
| **Scan timeout** | none — a wedged document hangs forever | every scan is time-boxed; the stuck worker is killed and respawned |
| **Backpressure** | requests pile up unbounded | when all workers are busy you get a clean "busy" error |
| **Input limits** | upload buffer can grow without bound | oversized / slow uploads are rejected, not buffered forever |
| **Self-healing** | restart the service on a cron | dead or mute workers are detected and replaced automatically |
| **Result cache** | — | optional redis cache: identical attachments are scanned once, shared across replicas |

The wire protocol is identical, so existing rspamd config keeps working — just
repoint it. `PING\n\n` still returns `PONG`.

---

## Quick start

```bash
docker run -d --name olefied \
  -e OLEFIED_WORKERS=4 \
  -p 10050:10050 \
  eilandert/olefied
```

Check it's alive:

```bash
printf 'PING\n\n' | nc -N 127.0.0.1 10050   # -> PONG
```

In a real mail stack you'd keep it on an internal network with **no published
port** and lock it down — see [Running it](#running-it) and [Security](#security).

---

## How it works

```
            :10050  (the only exposed port)
                │
        ┌───────▼─────────┐      loopback only, 127.0.0.1:10100+i
        │     olefyd       │ ───► olefy worker 0 ─► olevba
        │  dispatcher +    │ ───► olefy worker 1 ─► olevba
        │  supervisor      │ ───► olefy worker N ─► olevba
        └──────────────────┘
```

olefied listens on one public port and forwards each request to a free worker:

1. **Workers** — at startup it launches `OLEFIED_WORKERS` upstream olefy processes
   (default: one per CPU). Each listens on loopback only, with its own private
   scratch directory.
2. **Dispatch** — one in-flight scan per worker; idle workers are handed out from a
   queue. That gives fair scheduling and natural backpressure for free.
3. **Timeout & recovery** — each scan is bounded by `OLEFIED_REQUEST_TIMEOUT`. If a
   worker blows the limit it's assumed wedged on a poison document: olefied returns
   an error, kills the worker and spawns a fresh one.
4. **Supervision** — a background loop replaces workers that have died or gone
   unresponsive, so the pool heals itself.

---

## Connecting rspamd

Point rspamd's `oletools` external service at olefied — same config as for plain
olefy, just a different host:

```hcl
# external_services.conf  (or oletools.conf)
oletools {
  servers  = "olefied:10050";   # service name / VIP
  timeout  = 15s;
  max_size = 5M;
}
```

---

## Result cache (optional, redis)

`olevba` is the expensive part, and the **same attachments show up over and over**
in a mail stream: malware blasts, mailing-list footers, the quarterly report
everyone forwards. Scanning each one a thousand times to get the same answer is
wasted CPU.

Point olefied at redis and successful scans are cached by **document hash**. A
repeat of the same document skips scanning entirely, and a shared redis lets **all
replicas** reuse each other's results.

```yaml
services:
  olefied:
    image: eilandert/olefied
    environment:
      OLEFIED_REDIS_URL: redis://redis:6379/0
    networks: [ internal ]
  redis:
    image: redis:7-alpine
    networks: [ internal ]
```

How the cache behaves:

- **Key** = `prefix:v1:<oletools-version>:sha256(document-body)`. It hashes the
  **document only** — the per-message `Rspamd-ID` is ignored, so the same
  attachment in different mails is one entry. The oletools version is part of the
  key, so upgrading oletools automatically invalidates the old cache.
- **Only successful scans are cached.** Errors, timeouts and "busy" replies never
  are.
- **The cache can never break scanning.** If redis is down or slow, the lookup is
  treated as a miss and the scan just runs.

On real mail, expect a 30–70% hit rate — and every hit is a scan you didn't pay for.

---

## Benchmark

`olevba` is CPU-bound and a worker scans one document at a time, so per-container
throughput is roughly:

```
sustainable msg/s  ≈  OLEFIED_WORKERS / average_scan_seconds
```

A typical small attachment scans in ~50–200 ms, so a worker does ~5–20 msg/s and a
container scales with its core count. Replicas are stateless (no shared state, no
sticky sessions), so total throughput is the sum across replicas and grows linearly
until the CPUs saturate. The cache bends that line in your favour by removing scans
entirely.

Don't trust a number off a slide — **measure it on your own hardware and
documents.** Paste a representative attachment into `sample.bin` and run:

```python
# bench.py — python bench.py [host] [port] [concurrency] [requests]
import socket, sys, threading, time

HOST = sys.argv[1] if len(sys.argv) > 1 else "127.0.0.1"
PORT = int(sys.argv[2]) if len(sys.argv) > 2 else 10050
CONC = int(sys.argv[3]) if len(sys.argv) > 3 else 8
REQS = int(sys.argv[4]) if len(sys.argv) > 4 else 200

body = open("sample.bin", "rb").read()
req = b"OLEFY/1.0\nMethod: oletools\nRspamd-ID: bench0\n\n" + body
done, lat, lock = 0, [], threading.Lock()

def worker():
    global done
    while True:
        with lock:
            if done >= REQS:
                return
            done += 1
        t = time.perf_counter()
        s = socket.create_connection((HOST, PORT), 60)
        s.sendall(req); s.shutdown(socket.SHUT_WR)
        while s.recv(65536):
            pass
        s.close()
        with lock:
            lat.append(time.perf_counter() - t)

start = time.perf_counter()
ts = [threading.Thread(target=worker) for _ in range(CONC)]
[t.start() for t in ts]; [t.join() for t in ts]
dur = time.perf_counter() - start
lat.sort()
print(f"{len(lat)} scans, {CONC} concurrent, {dur:.1f}s "
      f"=> {len(lat)/dur:.1f} msg/s | "
      f"p50 {lat[len(lat)//2]*1000:.0f}ms p95 {lat[int(len(lat)*0.95)]*1000:.0f}ms")
```

Tips: set concurrency to roughly `OLEFIED_WORKERS` (more just queues), warm the
[result cache](#result-cache-optional-redis) first if you want hit-path latency, and
benchmark a single replica before extrapolating to the cluster.

To raise throughput: give the container more CPUs (`OLEFIED_WORKERS` defaults to the
CPU count), then add replicas behind a TCP load balancer.

---

## Running it

Hardened: read-only filesystem, no extra privileges, resource limits.

```bash
docker run -d --name olefied --init \
  --read-only --tmpfs /tmp:rw,mode=1777,size=512m \
  --cap-drop ALL \
  --security-opt no-new-privileges \
  --memory 1g --cpus 4 \
  -e OLEFIED_WORKERS=4 \
  -p 10050:10050 \
  eilandert/olefied
```

Or in a mail stack — internal network, no exposed port:

```yaml
services:
  olefied:
    image: eilandert/olefied
    init: true
    read_only: true
    tmpfs: [ "/tmp:mode=1777,size=512m" ]
    cap_drop: [ ALL ]
    security_opt: [ no-new-privileges:true ]
    environment:
      OLEFIED_WORKERS: "4"
      OLEFIED_REQUEST_TIMEOUT: "60"
    networks: [ internal ]
    deploy:
      replicas: 4          # scale this for throughput
      resources:
        limits: { cpus: "4", memory: 1g }
```

`--init` (or compose `init: true`) adds a PID-1 process reaper. It's belt-and-
suspenders — olefied already waits on the workers it spawns.

The image ships a `HEALTHCHECK` (a `PING`/`PONG` round-trip), so an orchestrator
won't route traffic to a container whose pool can't even answer a trivial request.

---

## Configuration

Everything is set through environment variables. `OLEFIED_*` tune the front-end;
`OLEFY_*` are passed straight through to the workers.

| Variable | Default | What it does |
|---|---|---|
| `OLEFY_BINDADDRESS` | `0.0.0.0` | Address olefied listens on |
| `OLEFY_BINDPORT` | `10050` | Port olefied listens on |
| `OLEFIED_WORKERS` | number of CPUs | How many olefy worker processes to run |
| `OLEFIED_REQUEST_TIMEOUT` | `60` | Max seconds for one scan; over this → error and the worker is recycled |
| `OLEFIED_READ_TIMEOUT` | `30` | Max seconds to read a client's upload |
| `OLEFIED_QUEUE_TIMEOUT` | `15` | Max seconds to wait for a free worker before returning "busy" |
| `OLEFIED_MAX_REQUEST_BYTES` | `52428800` | Reject uploads larger than this (50 MiB) |
| `OLEFIED_MAX_CONNS` | `workers × 4` (min 16) | Cap on simultaneous connections; bounds memory under a flood |
| `OLEFIED_HEALTH_INTERVAL` | `30` | How often (seconds) the supervisor checks workers |
| `OLEFIED_WORKER_BASE_PORT` | `10100` | First loopback port for workers |
| `OLEFY_LOGLVL` | `30` | Log level: 10 debug, 20 info, 30 warning, 40 error |
| `OLEFY_MINLENGTH` | `500` | olefy skips files smaller than this |
| `OLEFIED_REDIS_URL` | _(unset)_ | redis URL to enable the cache, e.g. `redis://redis:6379/0`; unset = cache off |
| `OLEFIED_CACHE_TTL` | `86400` | Cache entry lifetime in seconds (0 = forever) |
| `OLEFIED_CACHE_PREFIX` | `olefied` | redis key prefix |
| `OLEFIED_CACHE_OP_TIMEOUT` | `2` | Per-redis-operation timeout; a slow redis is treated as a cache miss |

Out-of-range values (zero, negative, non-numeric) are clamped to a safe minimum and
the correction is logged at startup, so a fat-fingered env var can't crash boot.

---

## Security

`olevba` parses **fully untrusted documents** — that's the whole job — so treat the
container as hostile territory:

- Runs as a **non-root** user, and works under `--read-only` with a `/tmp` tmpfs.
- **Multi-stage build:** no compiler or build tools in the final image — only the
  runtime, `libmagic1` and `netcat-openbsd`.
- Workers listen on **loopback only**; the dispatcher is the only thing exposed.
- In production: drop capabilities, keep it on an internal network, set memory/CPU
  limits, and update oletools deliberately for parser CVEs.

---

## How the image is built

`olefy.py` and its `requirements.txt` are **pulled fresh from upstream
([HeinleinSupport/olefy](https://github.com/HeinleinSupport/olefy)) at build time**
rather than vendored, so every build tracks Heinlein's latest. olefied sets the
`olevba` path explicitly in the image, so an upstream rename of that script (which
once broke the stock image) can't affect us.

For a reproducible build, pin a specific upstream revision:

```bash
docker build --build-arg OLEFY_REF=<tag-or-sha> -f docker/Dockerfile -t olefied .
```

---

## Credits & license

`olefy.py`, `olefy_ping.sh` and the test messages are © Heinlein Support, licensed
Apache-2.0 — see [HeinleinSupport/olefy](https://github.com/HeinleinSupport/olefy).
olefied's dispatcher and packaging are released under the same license.
