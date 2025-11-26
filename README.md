# Multicloud Kubernetes Federation POC

This repository contains a **proof of concept (PoC)** for a **multi-cluster / multi-region / multi-cloud** deployment on Kubernetes using:

- **KubeFed (Kubernetes Federation v2)** for multi-cluster resource propagation
- A simple HTTP app that prints its **hostname**, **region**, and **Pod IP**
- Optional **Istio + mTLS** as an _add-on_ to secure in-cluster traffic

The idea is that you deploy the same app to multiple clusters (for example **EKS** on AWS and **GKE** on GCP) and then verify:
- That the app is present in all clusters
- That each instance reports its own **region**
- That Federation is handling the distribution of workloads
- (Optional) That **Istio** enforces **mTLS** between services inside each cluster

> 🔒 Istio/mTLS is optional and documented separately in
> [`README-istio-addon.md`](README-istio-addon.md).

---

## Repository structure

Suggested structure for this repo:

```text
.
├── README.md                    # This file
├── README-istio-addon.md        # Istio + mTLS add-on (optional)
├── kubernetes
│   ├── kubefed-manifests.yaml   # FederatedNamespace, FederatedDeployment, FederatedService
│   └── istio-addon-manifests.yaml # Istio resources (PeerAuth, DestRule, Gateway, VS)
└── docs
    └── multicloud-architecture.drawio   # Architecture diagram (draw.io)
```

You can rename/move things as you like, but the examples in this README assume the structure above.

---

## Requirements

To run the PoC as described you’ll need:

### Tooling

- `kubectl` configured with **multiple clusters** (kubeconfig contexts)
- `kubefedctl` installed (for managing KubeFed)
- `helm` (optional, if you want to install KubeFed via Helm)
- `docker` or another container build tool, plus access to a container registry (GHCR, ECR, GCR, Docker Hub, etc.)

### Kubernetes clusters

At least **two Kubernetes clusters** reachable from your machine, e.g.:

- Cluster 1: AWS EKS in region `us-east-1`
- Cluster 2: GCP GKE in region `europe-west1`

You can adapt the names and regions as needed.

> In this README we’ll assume your KubeFed member clusters are named:
> - `aws-us-east-1`
> - `gcp-europe-west1`  
> (These names must match what you used with `kubefedctl join`.)

### KubeFed

You should have **KubeFed installed on a host cluster**, and the other clusters joined as members:

- Host cluster (where the KubeFed control plane lives)
- Member clusters: the clusters where the app should be propagated

High-level steps (not fully detailed here):

1. Install KubeFed (e.g. via Helm) into a host cluster
2. Use `kubefedctl join` to add:
   - `aws-us-east-1`
   - `gcp-europe-west1`
3. Verify with:
   ```bash
   kubectl -n kube-federation-system get kubefedclusters
   ```

Refer to the official KubeFed docs / mirrors for step-by-step installation if you don’t already have it.

### Container image for the demo app

Build and push the demo app image (Go service) included in this repo, or use your own.

Example (GitHub Container Registry):

```bash
# From the directory that contains main.go and Dockerfile
docker build -t ghcr.io/<your-org>/multicloud-demo:1.0.0 .

# Log in to GHCR (or your registry)
echo "$GITHUB_TOKEN" | docker login ghcr.io -u <your-username> --password-stdin

docker push ghcr.io/<your-org>/multicloud-demo:1.0.0
```

Then update the image reference in `kubernetes/kubefed-manifests.yaml`:

```yaml
image: ghcr.io/<your-org>/multicloud-demo:1.0.0
```

---

## What the app does

The demo app is a small HTTP server that:

- Listens on port `8080`
- Exposes:
  - `GET /` – returns JSON with:
    - `hostname` – Pod hostname
    - `region` – from the `REGION` environment variable
    - `podIP` – from the `POD_IP` environment variable (Downward API)
    - `time` – current UTC time
  - `GET /healthz` – always returns `200 OK`
  - `GET /readyz` – always returns `200 OK`

Example response from `GET /`:

```json
{
  "hostname": "multicloud-demo-6548f7ffbd-2s7gh",
  "region": "us-east-1",
  "podIP": "10.42.1.23",
  "time": "2025-01-01T12:34:56Z"
}
```

KubeFed overrides the `REGION` environment variable **per cluster**, so the response reflects the cluster where the Pod is running.

---

## KubeFed manifests (core POC)

The main KubeFed manifests live in:

- `kubernetes/kubefed-manifests.yaml`

This file includes:

1. `Namespace` + `FederatedNamespace` – `multicloud-demo` namespace, propagated to all member clusters.
2. `FederatedDeployment` – a federated Deployment for the app.
3. `FederatedService` – a federated Service for the app.

> These resources are applied only to the **host cluster** (where KubeFed runs).  
> KubeFed then creates the equivalent native resources in each **member cluster**.

### 1. Deploy the federated resources

On the **host cluster** (the one where KubeFed control plane is installed):

```bash
kubectl --context <host-cluster-context> apply -f kubernetes/kubefed-manifests.yaml
```

KubeFed will then:

- Create the `multicloud-demo` namespace in each member cluster
- Create the `Deployment` and `Service` in each member cluster
- Apply cluster-specific overrides:
  - `REGION=us-east-1` in the AWS cluster
  - `REGION=europe-west1` in the GCP cluster
  - (and different replica counts if desired)

### 2. Verify resources in each cluster

Check the Deployment and Service in each member cluster:

```bash
# AWS cluster
kubectl --context aws-us-east-1 -n multicloud-demo get deploy,svc

# GCP cluster
kubectl --context gcp-europe-west1 -n multicloud-demo get deploy,svc
```

You should see a `multicloud-demo` Deployment and Service in both.

### 3. Test the app (local port-forward)

For a quick test you can simply use `kubectl port-forward`:

```bash
# In AWS cluster
kubectl --context aws-us-east-1 -n multicloud-demo port-forward svc/multicloud-demo 8080:80

# In another terminal
curl http://localhost:8080/
curl http://localhost:8080/healthz
```

And for GCP:

```bash
# In GCP cluster
kubectl --context gcp-europe-west1 -n multicloud-demo port-forward svc/multicloud-demo 8080:80

# In another terminal
curl http://localhost:8080/
```

You should see different `region` values and likely different `hostname`/`podIP` per cluster.

### 4. (Optional) Expose with a LoadBalancer per cluster

If you want direct external access per cluster, create a simple LoadBalancer Service in **each** cluster (not federated), e.g.:

```bash
kubectl --context aws-us-east-1 -n multicloud-demo apply -f - <<'EOF'
apiVersion: v1
kind: Service
metadata:
  name: multicloud-demo-lb
spec:
  type: LoadBalancer
  selector:
    app: multicloud-demo
  ports:
    - name: http
      port: 80
      targetPort: 8080
EOF
```

Repeat similarly in `gcp-europe-west1`. Each cluster will get its external IP / hostname. You can then hit each endpoint to see its region/hostname.

---

## DNS configuration for multi-region (Route 53)

In a real multicloud / multiregion PoC you usually want **one global DNS name** for the app, for example:

- `demo.multicloud.example.com`

That name can be resolved to **different cluster entrypoints** (AWS / GCP) depending on the routing policy in **Amazon Route 53**.

Below are two common scenarios:

### Scenario 1: Latency-based routing (active/active)

Goal: send each client to the cluster/region with the **lowest latency**.

1. In **Route 53**, create (or use) a public hosted zone for your domain, e.g. `multicloud.example.com`.
2. For the record name `demo.multicloud.example.com`, create **two records** with **latency-based routing**:
   - Record 1 (AWS):
     - Type: `A` or ALIAS (if pointing to an AWS Load Balancer)
     - Region: `us-east-1`
     - Value: external IP or DNS name of the **AWS cluster entrypoint**, e.g.:
       - The Service `istio-ingressgateway` of the EKS cluster
       - Or an external LB in front of it
   - Record 2 (GCP):
     - Type: `A`
     - Region: (closest to GCP cluster, e.g. `EU` or a European region)
     - Value: external IP or DNS name of the **GCP cluster entrypoint** (GKE ingress / istio-ingressgateway LB).
3. (Recommended) Configure **Route 53 health checks** for each record:
   - Target: `https://demo.multicloud.example.com/healthz`
   - Endpoint: IP / DNS of each region’s LB.
4. With latency-based routing, Route 53 will return the record (AWS or GCP) that provides the **best latency** for each client **as long as the health check is healthy**.

In this setup:

- Both clusters are **active** (they can receive traffic at any time).
- Federation (KubeFed) ensures the app is deployed to all clusters.
- Route 53 decides where each user goes.

### Scenario 2: Failover routing (active/passive)

Goal: make **one cluster primary** (e.g. AWS us-east-1) and use another cluster (GCP) only as **backup** if the primary fails.

1. In your hosted zone `multicloud.example.com`, create **two records** for `demo.multicloud.example.com` with **failover routing**:
   - **PRIMARY record (AWS)**:
     - Type: `A` or ALIAS to the AWS LB / istio-ingressgateway.
     - Failover type: `PRIMARY`.
     - Attach a health check that probes `/healthz` on the AWS endpoint.
   - **SECONDARY record (GCP)**:
     - Type: `A` to the GCP LB / istio-ingressgateway.
     - Failover type: `SECONDARY`.
     - (Optionally) attach its own health check.
2. As long as the **PRIMARY** health check is healthy, Route 53 always returns the **AWS** record.
3. If the PRIMARY becomes **unhealthy**, Route 53 starts returning the **SECONDARY** record (GCP cluster).
4. When the PRIMARY is healthy again, Route 53 automatically fails back to AWS.

In this setup:

- AWS is **active**, GCP is **passive/DR** by DNS policy.
- KubeFed still deploys the app to all clusters, but only the PRIMARY endpoint receives traffic in normal conditions.

### Notes

- For a quick local test you can skip Route 53 and use `/etc/hosts` to point `demo.multicloud.example.com` directly to each cluster’s ingress IP.
- In production, combine:
  - Route 53 latency/failover policies,
  - Health checks against `/healthz`,
  - TLS (ALB / GKE / Istio gateway) as described in the manifests and the Istio add-on README.


---

## Optional add-on: Istio + mTLS

If you want to add **service mesh + mTLS** on top of the basic POC, see:

➡️ [`README-istio-addon.md`](README-istio-addon.md)

That add-on covers:

- Installing Istio in each cluster
- Enabling sidecar injection in `multicloud-demo` namespace
- Enforcing **mTLS** between services in the namespace
- Adding an Istio `Gateway` and `VirtualService` to expose the app via HTTPS

The manifests used by that add-on live in:

- `kubernetes/istio-addon-manifests.yaml`

KubeFed is still responsible for distributing the app’s Deployment/Service; Istio secures the **in-cluster** communication.

---

## Architecture diagram

A high-level architecture diagram (draw.io) is provided in:

- `docs/multicloud-architecture.drawio`

It shows:

- Two Kubernetes clusters (e.g. EKS and GKE)
- KubeFed host + member clusters
- The demo app deployed in both clusters
- Optional Istio ingress gateway if you apply the Istio add-on

You can open it with [draw.io / diagrams.net](https://app.diagrams.net/) and adjust it to match your actual environment.

---

## Cleaning up

To remove the POC resources, on the **host cluster**:

```bash
kubectl --context <host-cluster-context> delete -f kubernetes/kubefed-manifests.yaml
```

KubeFed will remove the propagated resources from all member clusters.

Any **non-federated** resources you created manually (e.g. `*-lb` Services, Istio resources from the add-on) must be deleted separately in each cluster.

---

## Notes & tips

- You can add more clusters (e.g. `aws-sa-east-1`, `gcp-us-central1`) by:
  - Joining them to KubeFed
  - Adding them to the `placement` section in `FederatedNamespace`, `FederatedDeployment`, `FederatedService`
  - Adding new `overrides` entries for their `REGION` env var
- For real production use, you probably want to combine this setup with:
  - **GitOps** (e.g. Argo CD / Flux) to manage the manifests
  - **Global DNS / routing** (e.g. Route 53 latency/failover, Cloudflare, etc.)
  - A proper CI pipeline to build and push the demo app image
