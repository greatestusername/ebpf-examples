# Phase 3: Kubernetes Deployment (Optional)

> **Goal**: Deploy the same three services to Kubernetes, add the OBI DaemonSet, and get full distributed tracing in Splunk APM -- same zero-code story, enterprise-grade orchestration.

This phase takes the exact same "naked" application code from Phase 2 and deploys it to a Kubernetes cluster. The OBI agent runs as a DaemonSet instead of a standalone container, instrumenting every pod on every node.

---

## Prerequisites


| Requirement                  | Why                                                                                                                                                                                   |
| ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Everything from Phase 1/2    | You need the app images and Splunk credentials                                                                                                                                        |
| A Kubernetes cluster         | Our workshops use [k3s](https://k3s.io/) on bare Linux instances. [k3d](https://k3d.io/), [kind](https://kind.sigs.k8s.io/), and [minikube](https://minikube.sigs.k8s.io/) also work. |
| `kubectl` configured         | Talking to your cluster                                                                                                                                                               |
| Docker (for building images) | Building the app container images                                                                                                                                                     |


---

## Step 0: Install k3s (if you don't have a cluster yet)

**NOTE:** Check if you have a cluster on your instance with `kubectl get nodes` if you have no cluster use these commands to install k3s

On a bare Linux instance (e.g. EC2), install k3s directly:

```bash
curl -sfL https://get.k3s.io | sh -
```

Wait a few seconds for the node to become `Ready`, then configure `kubectl`:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config

kubectl get nodes
```

> **k3s vs k3d**: k3s runs Kubernetes directly on the host. k3d wraps k3s inside Docker containers. On a bare instance, use k3s. On a laptop with Docker Desktop, use k3d.

---

## Step 1: Build and Load Images

The K8s manifests reference locally-built images. While in the `03-obi-k8s/` directory, build them from the `02-obi-docker/` source:

```bash
docker build -t obi-workshop-frontend:latest ../02-obi-docker/frontend
docker build -t obi-workshop-order-processor:latest ../02-obi-docker/order-processor
docker build -t obi-workshop-payment-service:latest ../02-obi-docker/payment-service
```

Import them into your cluster's container runtime:

**k3s (bare instance -- default for workshops):**

k3s uses containerd, not Docker, so images must be exported from Docker and imported:

```bash
docker save obi-workshop-frontend:latest | sudo k3s ctr images import -
docker save obi-workshop-order-processor:latest | sudo k3s ctr images import -
docker save obi-workshop-payment-service:latest | sudo k3s ctr images import -
```

### Working with other K8s architectures

**k3d / kind / minikube**

**k3d:**

```bash
k3d image import obi-workshop-frontend:latest obi-workshop-order-processor:latest obi-workshop-payment-service:latest
```

**kind:**

```bash
kind load docker-image obi-workshop-frontend:latest obi-workshop-order-processor:latest obi-workshop-payment-service:latest
```

**minikube:**

```bash
minikube image load obi-workshop-frontend:latest
minikube image load obi-workshop-order-processor:latest
minikube image load obi-workshop-payment-service:latest
```

> If you're using a remote cluster (EKS, GKE, etc.), push images to a registry and update `image:` fields in `apps.yaml`. Remove the `imagePullPolicy: Never` lines.

## Step 2: Add Your Splunk Credentials

Open `collector.yaml` and replace the four `<REPLACE_ME>` values in the `env` section:

```yaml
env:
  - name: SPLUNK_INGEST_TOKEN
    value: "YOUR_TOKEN_HERE"
  - name: SPLUNK_REALM
    value: "us0"
  - name: WORKSHOP_HOST_NAME
    value: "jsmith-k8s"
  - name: WORKSHOP_ENVIRONMENT
    value: "jsmith-k8s-workshop"
```

Save the file.

## Step 3: Deploy the Baseline (No Tracing)

Apply the manifests in order:

```bash
kubectl apply -f namespace.yaml
kubectl apply -f collector-configmap.yaml
kubectl apply -f collector.yaml
kubectl apply -f apps.yaml
kubectl apply -f load-generator.yaml
```

Verify everything is running:

```bash
kubectl get pods -n obi-workshop
```

All pods should be `Running`. Access the frontend via port-forward:

```bash
kubectl port-forward -n obi-workshop svc/frontend 3000:3000 &
```

Open [http://localhost:3000](http://localhost:3000) and click "Create Order", or if you're on a headless instance:

```bash
curl -s http://localhost:3000/create-order | python3 -m json.tool
```

### Check Splunk

- **Infrastructure**: You should see host metrics with your `host.name`.
- **APM**: Should be **empty** -- no traces, no services. Same as Phase 1.

---

## Step 4: Add OBI (The DaemonSet)

Now add tracing to the entire cluster without changing any application code.

```bash
kubectl apply -f obi-daemonset.yaml
```

This creates:

1. A **ConfigMap** with the OBI service discovery config (same as `obi-config.yaml` from Phase 2)
2. A **DaemonSet** that runs one OBI pod on every node in your cluster

### What Does the DaemonSet Do?

```yaml
hostPID: true        # See all processes on the node, including other pods
hostNetwork: true    # Observe and inject trace context into network traffic
privileged: true     # Attach eBPF probes to the kernel
```

OBI discovers your services by port (3000, 8080, 8081), instruments them via eBPF, generates traces, and sends them to the collector at `splunk-otel-collector.obi-workshop:4318`.

### Verify OBI is Running

```bash
kubectl get pods -n obi-workshop -l app=obi
kubectl logs -n obi-workshop -l app=obi --tail=20
```

You should see logs like:

```
time=2026-02-26T19:25:22.060Z level=WARN msg="OBI will still work, but features depending on pinned maps (e.g., log enricher, profile correlation) will be disabled"
time=2026-02-26T19:25:22.536Z level=INFO msg="instrumenting process" component=discover.traceAttacher cmd=/usr/local/bin/payment-service pid=188700 ino=3098470 type=go service=payment-service logenricher=false
time=2026-02-26T19:25:22.546Z level=INFO msg="instrumenting process" component=discover.traceAttacher cmd=/app/arcade pid=60565 ino=2636705 type=go service=order-processor logenricher=false
time=2026-02-26T19:25:22.584Z level=INFO msg="instrumenting process" component=discover.traceAttacher cmd=/usr/local/bin/order-processor pid=188612 ino=3098463 type=go service=order-processor logenricher=false
```

### Check Splunk APM

Wait 30-60 seconds, then check Splunk APM. You should see:

1. **Service Map**: `frontend` -> `order-processor` -> `payment-service`
2. **Traces**: Full distributed traces spanning all three services
3. **Same story as Phase 2**: Zero code changes, one DaemonSet addition

---

## How This Scales

In Phase 2 (Docker), you added one container. In Phase 3 (K8s), you added one DaemonSet. The pattern is the same:


| Environment    | OBI Deployment    | What Changes                                     |
| -------------- | ----------------- | ------------------------------------------------ |
| Bare host      | Binary via `sudo` | Nothing -- OBI watches processes from the kernel |
| Docker Compose | One container     | Add a service to `docker-compose.yaml`           |
| Kubernetes     | One DaemonSet     | `kubectl apply` a DaemonSet manifest             |


For production, you would also consider the [OpenTelemetry Operator](https://opentelemetry.io/docs/kubernetes/operator/) which can inject OBI as a sidecar automatically using annotations -- no DaemonSet needed.

---

## Cleanup

```bash
kubectl delete namespace obi-workshop
```

---

## Back to Workshop

- [Phase 0: Python Warm-up](../01-obi-python/README.md)
- [Phases 1 & 2: Docker](../02-obi-docker/README.md)
- [Extend with an LLM](../LLM.md)

