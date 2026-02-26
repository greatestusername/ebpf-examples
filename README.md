# OBI Workshop: Zero-Code APM with eBPF

A hands-on workshop demonstrating how **OpenTelemetry eBPF Instrumentation (OBI)** adds full distributed tracing to applications **without changing a single line of code**.

This workshop evolves from a single Python process on a bare Linux host to a polyglot Docker microservices architecture -- all instrumented by OBI from the kernel, with telemetry flowing to Splunk Observability Cloud.

---

## Workshop Structure

| Phase | Directory | What You Do |
|---|---|---|
| **Phase 0** -- Warm-up | [`01-obi-python/`](01-obi-python/) | Run a Python Flask app on the host. Send a heartbeat metric to Splunk. Download and run the OBI binary to instrument the process from the kernel. |
| **Phase 1** -- Before OBI | [`02-obi-docker/`](02-obi-docker/) | Deploy 3 microservices (Node.js + Go + Go) with Docker Compose. See infrastructure metrics in Splunk but **zero traces** in APM. |
| **Phase 2** -- The Magic | [`02-obi-docker/`](02-obi-docker/) | Add one OBI container to the compose file. Full distributed traces appear in Splunk APM across all three services. No code changes. |

```
Phase 0:  Python (:5000) ──── instrumented by OBI binary on host

Phase 1:  Frontend (Node.js :3000) → Order-Processor (Go :8080) → Payment-Service (Go :8081)
          ↑ infrastructure metrics only, APM is empty

Phase 2:  Same three services + one OBI container
          ↑ full distributed traces, zero code changes
```

---

## Prerequisites

| Requirement | Why |
|---|---|
| A **Linux** host (or Docker Desktop on Mac/Windows) | eBPF requires the Linux kernel |
| Python 3.9+ | Phase 0 warm-up app |
| Docker & Docker Compose | Phases 1 & 2 |
| Splunk Observability Cloud account | Where metrics and traces are sent |
| Your **Splunk Access Token** (Ingest) | [Org Settings > Access Tokens](https://app.signalfx.com/#/organization/tokens) |
| Your **Splunk Realm** (e.g. `us0`, `us1`, `eu0`) | [Shown in your Splunk Observability URL](https://docs.splunk.com/observability/en/admin/references/organizations.html) |
| A **unique name** (e.g. `jsmith-laptop`) | Used as `host.name` so you can find your own telemetry |

---

## Getting Started

1. **Fork** this repo (click the **Fork** button at the top-right of the repo page).

2. **Clone** your fork:

```bash
git clone https://github.com/<YOUR_GITHUB_USERNAME>/ebpf-examples.git
cd ebpf-examples
```

3. **Follow each phase in order:**
   - [Phase 0: Warm-up on Host](01-obi-python/README.md)
   - [Phases 1 & 2: Docker Multi-Service](02-obi-docker/README.md)

---

## This is a Workshop, Not a Demo

This is a **teaching tool**. Participants physically edit files at each phase:

- **Phase 0**: Replace `<REPLACE_ME>` placeholders with Splunk credentials. Download and run the OBI binary.
- **Phase 1**: Replace `<REPLACE_ME>` placeholders in `docker-compose.yaml`. Start the stack. Confirm APM is empty.
- **Phase 2**: Copy-paste the OBI service block from the README into the compose file. Watch traces appear.

An answer key ([`docker-compose.final.yaml`](02-obi-docker/docker-compose.final.yaml)) is provided if you get stuck.

---

## Extending the Workshop

Once you've completed all phases, see [LLM.md](LLM.md) for ideas on using an LLM (Cursor, Copilot, ChatGPT, etc.) to add new endpoints, services, error scenarios, and more.
