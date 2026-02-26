# Project: OBI (OpenTelemetry eBPF Instrumentation) Workshop Demo

## Role

Act as a Staff/Principal Observability Engineer. Build a "Zero-Code" workshop demo showcasing OpenTelemetry eBPF Instrumentation (OBI) sending data to Splunk Observability Cloud.

## Objective

Create a multi-service application that demonstrates "Before" (No APM) and "After" (Full APM via OBI) scenarios using Docker Compose. 

- Secondary Objective: These application will serve as an example app folks can tinker with using an LLM to add new features or endpoints or whatever. We should include a document details how to do that/what to look out for.

## Architecture (The "Polyglot" Chain)

1. **Frontend (Node.js):** An Express.js app. It has one interaction on the page to `/create-order` which makes an HTTP GET request to the Order-Processor. User should also be able to go to the page and "create an order" to create traffic manually
2. **Order-Processor (Go):** A compiled Go binary. It receives the request and calls the Payment-Service.
3. **Payment-Service (Go):** A compiled Go binary. It returns a JSON success response.
4. **Splunk OTel Collector:** The central pipeline for all telemetry.
5. **OBI Agent:** The `otel-ebpf-instrumentation` container that will instrument all three services out-of-process.

## Phase 1: The "Before" State

- Create `docker-compose.before.yaml`.
- Include the 3 services and the Splunk OTel Collector.
- **Goal:** The services should run, but NO tracing headers or OTel SDKs should be present in the code. In Splunk, the user should only see "Host/Container" metrics (CPU/RAM/ETC) with proper attributes for [host.name](http://host.name) and so on. Also a single CUSTOM METRIC for heartbeat but an empty APM dashboard.

## Phase 2: The "After" State (The OBI Magic)

- Create `docker-compose.after.yaml`.
- Add the `otel-ebpf-instrumentation` service.
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
  - Configure the Collector to export to Splunk Observability using the `otlp` (traces GET THE CORRECT ENDPOINT FROM THE DOCS) and `signalfx` (metrics) exporters.
3. **Load Generator:** Include a simple `curl` loop in a separate container to keep the "Order" flow active.

## Deliverables

1. `/frontend` (Node.js source).
2. `/order-processor` (Go source).
3. `/payment-service` (Go source).
4. `collector-config.yaml` (Splunk-specific).
5. `docker-compose.before.yaml` and `docker-compose.after.yaml`.
6. `README.md` with:
  - Setup instructions for Splunk credentials.
  - A "Sales Architect Pitch" explaining that OBI provides visibility into legacy or compiled apps where developers refuse (or are unable) to add code.
  - Sub section (or separate file really like LLM.md) that suggests how users of the workshop can play with an LLM to add endpoints or other telemetry data/things to this project in a fork (SECONDARY OBJECTIVE)

## Hands-On Workshop Requirement

This is a teaching tool, not a click-to-run demo. SAs must physically modify the `docker-compose.yaml` at each phase:

- **Phase 1**: Replace credential placeholders (`<REPLACE_ME>`) with their Splunk token and realm directly in the compose file.
- **Phase 2**: Copy-paste the OBI service block from the README instructions into the compose file. The README explains EACH line of the OBI config so SAs understand what they're adding and can replicate it for customers.

A single `docker-compose.yaml` is used (not separate before/after files). SAs evolve it through the workshop. An "answer key" with the final state is provided at the bottom of the README for reference.