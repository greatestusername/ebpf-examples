# Extending This Workshop with an LLM

This project is designed to be a playground. Once you've completed the workshop, use an LLM (Cursor, GitHub Copilot, ChatGPT, etc.) to modify and extend it.

## Ideas to Try

### Add a New Endpoint

Ask your LLM:

> "Add a `GET /order-status/:id` endpoint to the order-processor that returns a fake status. Make the frontend have a button that calls it."

OBI will automatically trace the new endpoint -- you don't need to change any OBI config unless you add a new service on a new port.

### Add a New Service

Ask your LLM:

> "Add an inventory-service in Python (Flask) on port 8082. Have the order-processor call it before calling payment-service. Update the docker-compose and OBI config."

After doing this, you'll need to:
1. Add the new service to `docker-compose.yaml`
2. Add a new entry to `obi-config.yaml`:
   ```yaml
   - name: "inventory-service"
     open_ports: 8082
   ```
3. `docker compose up --build -d` to pick up changes

OBI will instrument the Python service the same way it instruments Go and Node.js.

### Add Error Scenarios

Ask your LLM:

> "Make the payment-service randomly fail 20% of the time with a 500 error. Add retry logic to the order-processor."

This creates interesting traces in Splunk APM -- you'll see error spans and retry patterns.

### Add Latency Simulation

Ask your LLM:

> "Add random latency (100-500ms) to the payment-service. I want to see it in Splunk APM trace waterfall."

## Things to Watch Out For

### Do NOT add OpenTelemetry SDKs

The entire point of this workshop is zero-code instrumentation. If your LLM suggests adding `@opentelemetry/sdk-node` or `go.opentelemetry.io/otel`, reject it. OBI handles all the tracing.

If you want to explore SDK-based instrumentation as a comparison exercise, create a separate branch.

### Keep Services on the Docker Network

New services need to be reachable by other services via Docker Compose DNS (e.g. `http://inventory-service:8082`). Don't use `localhost` for inter-service communication in the application code -- `localhost` inside a container refers to that container, not the host.

### Update obi-config.yaml for New Ports

OBI discovers services by port. If you add a new service on a new port, add a matching entry to `obi-config.yaml` so OBI knows to instrument it and what to call it.

### Rebuild After Code Changes

After modifying source code:

```bash
docker compose up --build -d
```

The `--build` flag tells Docker Compose to rebuild images that have changed source files.

## Suggested LLM Prompts

Here are some prompts that work well with this codebase:

1. "Look at the three services in this repo. Add a `GET /health-details` endpoint to each that returns the service name, uptime, and current timestamp."

2. "Add a Redis cache to this stack. Have the order-processor cache payment results for 30 seconds. Add Redis to docker-compose.yaml and update obi-config.yaml."

3. "Create a simple dashboard page at `GET /dashboard` on the frontend that shows the last 10 orders. Store them in memory."

4. "Add request logging middleware to all three services that logs method, path, status code, and duration. Use only stdlib -- no external logging libraries."
