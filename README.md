# OBI Workshop: Zero-Code APM with eBPF

A hands-on workshop demonstrating how **OpenTelemetry eBPF Instrumentation (OBI)** adds full distributed tracing to applications **without changing a single line of code**.

You will run three "naked" microservices (no OTel SDKs, no tracing libraries), see that they produce zero APM data, then add one container and watch full traces appear in Splunk APM.

```
Frontend (Node.js :3000)  -->  Order-Processor (Go :8080)  -->  Payment-Service (Go :8081)
         \                                                              /
          `------- all instrumented by OBI via eBPF, zero code -------'
```

---

## Prerequisites


| Requirement                                           | Why                                                                                                                    |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Docker & Docker Compose                               | Runs the services, collector, and OBI agent                                                                            |
| A **Linux** host (or Docker Desktop on Mac/Windows)   | eBPF requires the Linux kernel; Docker Desktop runs a Linux VM under the hood                                          |
| Splunk Observability Cloud account                    | Where metrics and traces are sent                                                                                      |
| Your **Splunk Access Token** (Ingest)                 | [Org Settings > Access Tokens](https://app.signalfx.com/#/organization/tokens)                                         |
| Your **Splunk Realm** (e.g. `us0`, `us1`, `eu0`)      | [Shown in your Splunk Observability URL](https://docs.splunk.com/observability/en/admin/references/organizations.html) |
| A **unique name** for yourself (e.g. `jsmith-laptop`) | Used as `host.name` so you can find your own telemetry in Splunk                                                       |


## Getting Started

1. **Fork** this repo to your own GitHub account (click the **Fork** button at the top-right of the repo page).
2. **Clone** your fork:

```bash
git clone https://github.com/<YOUR_GITHUB_USERNAME>/OBI-eBPF-example.git
cd OBI-eBPF-example
```

You'll be editing files directly during the workshop, and having your own fork means you can push your changes and experiment freely.

---

## Phase 1: The "Before" State

> **Goal**: Get the services running and sending infrastructure metrics to Splunk. Confirm that APM is empty -- no traces exist because there is no instrumentation.

### Step 1: Add Your Splunk Credentials and Identity

Open `docker-compose.yaml` in your editor. Find the `splunk-otel-collector` service and replace the four `<REPLACE_ME>` placeholders with your real values:

```yaml
  splunk-otel-collector:
    image: quay.io/signalfx/splunk-otel-collector:latest  # docs: https://docs.splunk.com/observability/en/gdi/opentelemetry/opentelemetry.html
    # ...
    environment:
      SPLUNK_INGEST_TOKEN: "YOUR_TOKEN_HERE"       # <-- Your Splunk ingest token
      SPLUNK_REALM: "us0"                          # <-- Your realm (us0, us1, eu0, etc.)
      WORKSHOP_HOST_NAME: "jsmith-laptop"          # <-- Your name/initials + machine (must be unique!)
      WORKSHOP_ENVIRONMENT: "jsmith-workshop"      # <-- A unique label so you can filter in Splunk
      SPLUNK_CONFIG: /etc/otel/config.yaml
```

**Why `WORKSHOP_HOST_NAME` and `WORKSHOP_ENVIRONMENT`?** Everyone in the workshop sends telemetry to the same Splunk org. These values become the `host.name` and `deployment.environment` attributes on all your metrics and traces, so you can filter to **your** data in Splunk and not see everyone else's.

Save the file.

### Step 2: Start the Stack

```bash
docker compose up --build -d
```

This builds the three application images from source and starts everything:

- **frontend** on [http://localhost:3000](http://localhost:3000)
- **order-processor** on port 8080
- **payment-service** on port 8081
- **splunk-otel-collector** receiving telemetry on ports 4317/4318
- **load-generator** automatically hitting `/create-order` every 2 seconds

### Step 3: Generate Traffic Manually

Open [http://localhost:3000](http://localhost:3000) in your browser and click **"Create Order"** a few times. You should see a JSON response like:

```json
{
  "order": "confirmed",
  "payment": {
    "status": "success",
    "transaction_id": "txn-a1b2c3d4e5f6",
    "amount": 42
  }
}
```

The request flowed through all three services. But right now, nobody is watching.

### Step 4: Check Splunk Observability

1. **Infrastructure Monitoring**: The `workshop.heartbeat` custom metric should also be ingested. Search for it in [Metric Finder](https://shw-playground.signalfx.com/#/metrics). You should see `host.name` that matches yours as an attribute of that metric. Search for that `host.name` (or any other attached to that metric) in the Metric Finder to see what other metrics you are currently sending.
2. **APM**: Navigate to APM and try to filter to the environment you defined earlier. It should be **empty (or not exist)**. No services, no traces, no service map.

### Step 5: Look at the Code

Take a moment to inspect the source code:

- `frontend/server.js` -- plain Express.js, no imports from `@opentelemetry/`*
- `order-processor/main.go` -- standard `net/http`, no OTel dependencies
- `payment-service/main.go` -- standard `net/http`, no OTel dependencies

There are **zero tracing headers, zero SDKs, zero instrumentation** anywhere. 

The collector is sending infrastructure metrics because that's what it does by default, but it has no traces to export because nothing is generating them.

**Now let's fix that -- without touching the application code.**

---

## Phase 2: The OBI Magic

> **Goal**: Add the OBI eBPF agent to your compose file. Without changing any application code, full distributed traces will appear in Splunk APM.

### What is OBI?

OBI ([OpenTelemetry eBPF Instrumentation](https://opentelemetry.io/docs/zero-code/obi/)) is a standalone agent that uses Linux kernel eBPF probes to observe HTTP/gRPC traffic flowing through applications. 

It attaches to processes **from the kernel!**  
No SDK, no code changes, no recompilation. It sees the requests, generates OpenTelemetry-compatible traces, and sends them to a collector.

### Step 1: Add the OBI Service to docker-compose.yaml

> Full docs: [Run OBI as a Docker container](https://opentelemetry.io/docs/zero-code/obi/setup/docker) | [Docker Hub image](https://hub.docker.com/r/otel/ebpf-instrument)

Open `docker-compose.yaml` in your editor. Scroll to the **very bottom** of the file -- you'll see a comment block that says `PHASE 2`. Paste the following block **directly below that comment**, keeping the **2-space indentation** so it lines up with the other services (like `frontend:`, `load-generator:`, etc.):

```yaml
  obi:
    image: otel/ebpf-instrument:main
    pid: host
    privileged: true
    network_mode: host
    volumes:
      - ./obi-config.yaml:/config/obi-config.yaml
      - /sys/fs/cgroup:/sys/fs/cgroup
    environment:
      OTEL_EBPF_CONFIG_PATH: /config/obi-config.yaml
```

> **Common mistake**: Make sure `obi:` is indented 2 spaces (same level as `frontend:`, `load-generator:`, etc.). If it's at the far left with no indentation, Docker Compose will reject it with `Additional property obi is not allowed`. It must be **inside** the `services:` block, not above or outside it.

Save the file.

### What Does Each Line Do?

> Deep dive: [OBI global config options](https://opentelemetry.io/docs/zero-code/obi/configure/options)


| Line                               | What it does                                                                             | Why it matters                                                                                                                                                                                                            |
| ---------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `image: otel/ebpf-instrument:main` | The [OBI container image](https://hub.docker.com/r/otel/ebpf-instrument) from Docker Hub | This is the only thing you're adding to your stack                                                                                                                                                                        |
| `pid: host`                        | Shares the host's PID namespace                                                          | OBI needs to see processes running in **other** containers. Without this, it can only see its own process                                                                                                                 |
| `privileged: true`                 | Grants kernel-level access                                                               | eBPF programs need to attach probes to kernel functions. This is a [hard requirement](https://opentelemetry.io/docs/zero-code/obi/setup/docker)                                                                           |
| `network_mode: host`               | Shares the host's network stack                                                          | Required for [context propagation](https://opentelemetry.io/docs/zero-code/obi/configure/metrics-traces-attributes/#context-propagation) -- OBI injects trace context at the network level so traces link across services |
| `volumes: ./obi-config.yaml:...`   | Mounts the service discovery config                                                      | Tells OBI which processes to instrument and what to name them                                                                                                                                                             |
| `volumes: /sys/fs/cgroup:...`      | Mounts the cgroup filesystem                                                             | OBI uses this to detect which processes are running inside containers                                                                                                                                                     |
| `OTEL_EBPF_CONFIG_PATH`            | Points to the config file inside the container                                           | Standard OBI env var for [configuration](https://opentelemetry.io/docs/zero-code/obi/configure/options)                                                                                                                   |


### Step 2: Understand the OBI Config

> Full docs: [Service discovery](https://opentelemetry.io/docs/zero-code/obi/configure/service-discovery) | [Context propagation](https://opentelemetry.io/docs/zero-code/obi/configure/metrics-traces-attributes/#context-propagation) | [Config example](https://opentelemetry.io/docs/zero-code/obi/configure/example/)

Open `obi-config.yaml` (already in the repo). Here's what each section does:

```yaml
discovery:
  instrument:
    - name: "frontend"          # Service name that appears in Splunk APM
      open_ports: 3000          # "Find the process listening on port 3000"
    - name: "order-processor"
      open_ports: 8080
    - name: "payment-service"
      open_ports: 8081
```

[discovery.instrument](https://opentelemetry.io/docs/zero-code/obi/configure/service-discovery) tells OBI how to find your services and what to name them. It matches processes by the ports they listen on, then assigns the `name` as the `service.name` attribute in the generated traces. Without this, OBI would use the executable path as the service name (e.g. `/usr/local/bin/order-processor`).

```yaml
ebpf:
  context_propagation: all
```

[context_propagation: all](https://opentelemetry.io/docs/zero-code/obi/configure/metrics-traces-attributes/#context-propagation) is the key to distributed tracing. OBI injects `Traceparent` headers into outgoing HTTP requests at the kernel level. This is how a trace started in `frontend` connects through `order-processor` to `payment-service` -- even though none of these services know anything about tracing.

```yaml
otel_traces_export:
  endpoint: http://localhost:4318
```

`otel_traces_export.endpoint` tells OBI where to send traces. Because OBI uses `network_mode: host`, `localhost:4318` reaches the collector's port that we mapped to the host in the compose file.

### Step 3: Start OBI

```bash
docker compose up -d
```

Docker Compose will detect that only the `obi` service is new and start it. Your existing services keep running.

### Step 4: Check Splunk APM

Wait 30-60 seconds for traces to flow, then check Splunk APM:

1. **Service Map**: You should now see three services: `frontend` -> `order-processor` -> `payment-service`
2. **Traces**: Click into any trace. You'll see the full distributed trace spanning all three services with timing for each hop.
3. **Compare to Phase 1**: The APM dashboard that was completely empty 5 minutes ago now shows a full service topology.

**You added ONE container to your compose file. You changed ZERO lines of application code. You now have full distributed tracing.**

### Verification Checklist

```bash
# All containers should be running
docker compose ps

# This should return a JSON order confirmation
curl localhost:3000/create-order

# Check OBI logs to see it found your services
docker compose logs obi | head -30
```

- `docker compose ps` shows all 6 containers running
- `curl localhost:3000/create-order` returns JSON
- Splunk Infrastructure shows host metrics
- Splunk APM shows the 3-service trace chain

---

## The Sales Pitch

### The Problem

Many organizations have applications they **cannot** or **will not** instrument with OpenTelemetry SDKs:

- **Legacy systems**: COBOL-to-Java migrations, decade-old .NET Framework apps, vendor-provided binaries with no source access
- **Compiled languages**: Go, Rust, C++ services where recompilation isn't an option or the team has moved on
- **Developer resistance**: "We don't have time", "It's not in the sprint", "We're not changing working code"
- **Regulatory constraints**: Any code change triggers a full audit/certification cycle

### The OBI Answer

OBI gives you **full distributed tracing without any code changes**:

- **Zero SDK integration** -- no imports, no dependencies, no compile-time changes
- **Zero application restarts** -- OBI attaches to already-running processes via eBPF
- **Language agnostic** -- works with Go, Node.js, Python, Java, Rust, C++ -- anything that speaks HTTP or gRPC
- **One container** -- add it to your compose/K8s manifest and you're done

### The Demo Talking Points

1. "Look at the source code. There is literally nothing observability-related in it."
2. "In Phase 1, we had metrics but zero traces. The APM dashboard was empty."
3. "All I did was add one container to the compose file. I didn't touch the application code, I didn't rebuild anything, I didn't restart the services."
4. "Now I have full distributed traces showing every request flowing through three services written in two different languages."
5. "This is what we can do for your legacy services tomorrow."

---

## Cleanup

```bash
docker compose down
```

---

## Answer Key

If you got stuck, here is the complete final `docker-compose.yaml` with all changes applied (credentials and OBI service):

```yaml
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - order-processor

  order-processor:
    build: ./order-processor
    ports:
      - "8080:8080"
    depends_on:
      - payment-service

  payment-service:
    build: ./payment-service
    ports:
      - "8081:8081"

  splunk-otel-collector:
    image: quay.io/signalfx/splunk-otel-collector:latest
    ports:
      - "4317:4317"
      - "4318:4318"
      - "13133:13133"
    volumes:
      - ./collector-config.yaml:/etc/otel/config.yaml
    environment:
      SPLUNK_INGEST_TOKEN: "YOUR_TOKEN_HERE"
      SPLUNK_REALM: "us0"
      WORKSHOP_HOST_NAME: "jsmith-laptop"
      WORKSHOP_ENVIRONMENT: "jsmith-workshop"
      SPLUNK_CONFIG: /etc/otel/config.yaml

  load-generator:
    image: curlimages/curl:latest
    depends_on:
      - frontend
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        echo "Waiting for frontend to start..."
        sleep 10
        echo "Starting load generation..."
        while true; do
          curl -s http://frontend:3000/create-order
          sleep 2
        done

  obi:
    image: otel/ebpf-instrument:main
    pid: host
    privileged: true
    network_mode: host
    volumes:
      - ./obi-config.yaml:/config/obi-config.yaml
      - /sys/fs/cgroup:/sys/fs/cgroup
    environment:
      OTEL_EBPF_CONFIG_PATH: /config/obi-config.yaml
```

