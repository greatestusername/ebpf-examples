# Phase 0: The "Naked" Warm-up (Host Level)

> **Goal**: Run a bare Python app on the host, prove connectivity to Splunk with a custom metric, then use the OBI binary to add APM tracing -- all without Docker.

This phase shows that OBI works at the **kernel level** on a raw Linux process. No containers, no sidecars, no SDKs -- just an eBPF binary watching your app from the kernel.

> **Platform requirement**: eBPF is a Linux kernel feature. This step **requires** a **Linux host** (native or VM). It will **not** work on *macOS* or *Windows* -- Docker Desktop runs a Linux VM that is isolated from your host OS, so OBI inside Docker cannot see processes running on macOS/Windows. If you're on macOS/Windows, skip ahead to [Phase 1 & 2](../02-obi-docker/README.md) where OBI runs alongside the apps in the same Docker environment.

---

## Step 1: Install Dependencies

You need Python 3.9+ on the host.

```bash
cd 01-obi-python
python3 -m venv .venv
source .venv/bin/activate
pip3 install -r requirements.txt
```

## Step 2: Set Your Splunk Credentials

Export your credentials as environment variables. Replace each `<REPLACE_ME>`:

```bash
export SPLUNK_INGEST_TOKEN="<REPLACE_ME>"      # Your Splunk ingest token
export SPLUNK_REALM="<REPLACE_ME>"             # e.g. us0, us1, eu0
export WORKSHOP_HOST_NAME="<REPLACE_ME>"       # e.g. jsmith-laptop (unique to you)
```

## Step 3: Run the App in the background so we can send request to it

```bash
nohup python3 app.py &
```

On startup the app sends a single `app.heartbeat` metric directly to the Splunk Ingest API via HTTP POST. You should see:

```
Heartbeat sent to Splunk (200)
 * Running on http://0.0.0.0:5150
```

Hit the endpoint to confirm it's working from your browser or the below curl command:

```bash
curl http://localhost:5150/hello
```

### Verify in Splunk

1. Open [Metric Finder](https://app.signalfx.com/#/metrics) and search for `app.heartbeat`.
2. You should see the metric with `host.name` matching the value you set.

**At this point you have a running app and proof that Splunk can receive your data. But there are zero traces -- APM is empty.**

---

## Step 4: Instrument with the OBI Binary

Now add APM tracing to this running app **without touching a single line of code**.

> Full docs: [OBI standalone setup](https://opentelemetry.io/docs/zero-code/obi/setup/standalone/)

### Option A: Run the OBI Binary Directly (Linux Only)

Extract the binary from the Docker image (no standalone downloads yet:

```bash
IMAGE=otel/ebpf-instrument:main
sudo docker pull $IMAGE
ID=$(sudo docker create $IMAGE)
sudo docker cp "$ID:/obi" ./obi
sudo docker rm -v $ID

ls . #should see `obi` executable
```



In a **separate terminal**, run OBI with `sudo`. Replace the three `<REPLACE_ME>` values with your realm, token, and hostname from Step 2:

```bash
sudo OTEL_EXPORTER_OTLP_TRACES_ENDPOINT="https://ingest.<REPLACE_ME>.signalfx.com/v2/trace/otlp" \
     OTEL_EXPORTER_OTLP_HEADERS="X-SF-Token=<REPLACE_ME>" \
     OTEL_SERVICE_NAME="warmup-app" \
     OTEL_RESOURCE_ATTRIBUTES="deployment.environment=ebpf-bare-app,host.name=<REPLACE_ME>" \
     OTEL_EBPF_OPEN_PORT=5150 \
     ./obi
```


### Why These Flags/Variables?


| Flag / Variable | Purpose |
|---|---|
| `sudo` / `--privileged` | eBPF probes require root/kernel access |
| `--pid host` | (Docker only) Share the host PID namespace so OBI can see your Python process |
| `--network host` | (Docker only) Share the host network so OBI can observe traffic on port 5150 |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` | Full URL for Splunk's OTLP trace ingest. The per-signal env var sends to this URL exactly -- the base `OTEL_EXPORTER_OTLP_ENDPOINT` would append `/v1/traces` which doesn't match Splunk's path |
| `OTEL_EXPORTER_OTLP_HEADERS` | Auth header for Splunk |
| `OTEL_SERVICE_NAME` | The service name that appears in Splunk APM |
| `OTEL_RESOURCE_ATTRIBUTES` | Sets `deployment.environment` and `host.name` on every trace so you can filter to your data |
| `OTEL_EBPF_OPEN_PORT` | Tells OBI to instrument the process listening on port 5150 |

> **Note**: You may see warnings like `failed to upload metrics: 404 Not Found` in the OBI logs. This is expected -- Splunk's direct ingest doesn't have a standard OTLP metrics endpoint. The traces still export correctly. In Phase 2, a collector handles both traces and metrics properly.

### Generate Traffic

```bash
for i in $(seq 1 20); do curl -s http://localhost:5150/hello; sleep 1; done
```

### Verify in Splunk APM

1. Navigate to **APM** in Splunk Observability Cloud.
2. Filter by service name `warmup-app`.
3. You should see traces for the `/hello` endpoint.

**You just added distributed tracing to a running process from the kernel. No SDK, no code changes, no restart.**

---

## What Just Happened?

1. The Flask app is "naked" -- it has zero observability code. It only knows how to say hello and send a heartbeat metric.
2. OBI attached eBPF probes to the kernel's networking stack and observed HTTP traffic flowing through your app's process.
3. OBI generated OpenTelemetry-compatible trace spans and sent them directly to Splunk.

This is the same technology you'll use in Phase 1 & 2, but inside Docker containers instead of bare processes.

---

## Next Step

Proceed to [Phase 1 & 2: Docker Multi-Service](../02-obi-docker/README.md).