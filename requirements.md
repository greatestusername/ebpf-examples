# Project: OBI (OpenTelemetry eBPF Instrumentation) Workshop Demo

## Role

Act as a Staff/Principal Observability Engineer. Build a "Zero-Code" workshop demo showcasing OpenTelemetry eBPF Instrumentation (OBI) sending data to Splunk Observability Cloud.

## Objective

Create a multi-service application that demonstrates "Before" (No APM) and "After" (Full APM via OBI) scenarios. The workshop evolves from a single host-level process to a complex polyglot microservices architecture using Docker Compose.

- Secondary Objective: This application will serve as an example app folks can tinker with using an LLM to add new features or endpoints or whatever. We should include a document details how to do that/what to look out for.

## Architecture (The "Polyglot" Chain)

1. **Warm-up App (Python):** A single-service Flask app running directly on the Linux host (No Docker). 
    - This part (Phase 0) should have its own directory for organization. While Phases 1/2 and all their files also have their own directory for organizations
2. **Frontend (Node.js):** An Express.js app. It has one interaction on the page to `/create-order` which makes an HTTP GET request to the Order-Processor. User should also be able to go to the page and "create an order" to create traffic manually.
3. **Order-Processor (Go):** A compiled Go binary. It receives the request and calls the Payment-Service.
4. **Payment-Service (Go):** A compiled Go binary. It returns a JSON success response.
5. **Splunk OTel Collector:** The central pipeline for all telemetry.
6. **OBI Agent:** The `otel-ebpf-instrumentation` container that will instrument all three services out-of-process.


## Phase 0: The "Naked" Warm-up (Host Level)

- **The App:** Create a simple Python Flask app (`/warmup/app.py`) with a `/hello` endpoint.
- **No Docker:** This phase must run directly on the Linux host.
- **Manual Metric:** The app should manually send a single custom metric `app.heartbeat` to the Splunk Ingest API using a simple HTTP POST (to prove connectivity to Splunk).
- **The Task:** SAs run the Python app. They then download the OBI binary (OpenTelemetry Go Instrumentation) and run it via `sudo` to instrument the Python process.
- **Goal:** Show that OBI works at the kernel level on a raw process before moving to the "magic" of Docker.

## Phase 1: The "Before" State (Docker)

- Create `docker-compose.yaml` (Starter version).
- Include the 3 services (Node/Go/Go) and the Splunk OTel Collector.
- **Goal:** The services should run, but NO tracing headers or OTel SDKs should be present in the code. In Splunk, the user should only see "Host/Container" metrics (CPU/RAM/ETC) with proper attributes for host.name and so on. Also a single CUSTOM METRIC for heartbeat but an empty APM dashboard.

## Phase 2: The "After" State (The OBI Magic)

- SAs manually modify the `docker-compose.yaml` to add the `otel-ebpf-instrumentation` service.
- **OBI Configuration Requirements:**
  - Use the `otel-ebpf-instrumentation` image.
  - Run with `privileged: true` and `network_mode: host` (or shared PID namespaces).
  - Set `OTEL_EXPORTER_OTLP_ENDPOINT` to the Collector.
  - Use environment variables to define service names for each discovered process (e.g., `OTEL_SERVICE_NAME`).
- **Goal:** Once this container starts, traces must appear in Splunk APM showing the full flow: Node.js -> Go -> Go.

## Technical Requirements

1. **Zero Code Changes:** The Go and Node.js source code must NOT contain any OpenTelemetry SDKs or manual instrumentation. It should be "naked" code.
2. **Splunk Integration:**
  - Use `SPLUNK_INGEST_TOKEN` and `SPLUNK_REALM` env variables.
  - Configure the Collector to export to Splunk Observability using the `otlp` (traces: `https://ingest.${SPLUNK_REALM}.signalfx.com/v2/trace/otlp`) and `signalfx` (metrics: `https://ingest.${SPLUNK_REALM}.signalfx.com/v2/datapoint`) exporters.
3. **Load Generator:** Include a simple `curl` loop in a separate container to keep the "Order" flow active.

## Deliverables

1. `/01-obi-python` (Python source + instructions for running the OBI binary on host).
2. `/02-obi-docker/frontend` (Node.js source).
3. `/02-obi-docker/order-processor` (Go source).
4. `/02-obi-docker/payment-service` (Go source).
5. `/02-obi-dockercollector-config.yaml` (Splunk-specific).
6. `/02-obi-dockerdocker-compose.yaml` (Starter version with placeholders).
7. `/02-obi-dockerREADME.md` with:
  - Setup instructions for Splunk credentials.
  - A "Sales Architect Pitch" explaining that OBI provides visibility into legacy or compiled apps where developers refuse (or are unable) to add code.
  - Sub section (or separate file really like LLM.md) that suggests how users of the workshop can play with an LLM to add endpoints or other telemetry data/things to this project in a fork (SECONDARY OBJECTIVE).

## Hands-On Workshop Requirement

This is a teaching tool, not a click-to-run demo. SAs must physically modify the files at each phase:

- **Phase 0**: Replace credential placeholders (`<REPLACE_ME>`) with their Splunk token and endpoint and any OBI env vars download run obi to get apm data of the requests.
- **Phase 1**: Replace credential placeholders (`<REPLACE_ME>`) with their Splunk token and realm directly in the compose file.
- **Phase 2**: Copy-paste the OBI service block from the README instructions into the compose file. The README explains EACH line of the OBI config (e.g., why `privileged: true` is needed) so SAs understand what they're adding.

A single `docker-compose.yaml` is used. SAs evolve it through the workshop. An "answer key" with the final state (`docker-compose.final.yaml`) is provided for reference.