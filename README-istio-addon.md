# Istio mTLS Add-on for the Multicloud KubeFed POC

This document describes how to add **Istio** on top of the base POC to:

- Enable **mTLS** between services inside the `multicloud-demo` namespace
- Expose the demo app via an **Istio Ingress Gateway** over HTTPS

The base POC (KubeFed + demo app) is described in [`README.md`](README.md).

> ⚠️ This add-on is **optional**. The base POC works without Istio.

---

## High-level design

Per cluster (e.g. EKS and GKE):

1. Install **Istio** (profile `default`) → includes `istio-ingressgateway`
2. Enable **sidecar injection** in `multicloud-demo` namespace
3. Apply Istio policies:
   - `PeerAuthentication` with `mTLS: STRICT`
   - `DestinationRule` for `multicloud-demo` with `ISTIO_MUTUAL`
4. Create a TLS Secret for an HTTPS host (e.g. `demo.multicloud.example.com`)
5. Create a `Gateway` and `VirtualService` to route HTTPS traffic to the app
6. Ensure the `istio-ingressgateway` Service exposes port 443 as a `LoadBalancer`

Result:

- Client → HTTPS → Istio Ingress Gateway → **mTLS** → demo app Pods

---

## Requirements

- Base POC deployed:
  - `kubernetes/kubefed-manifests.yaml` applied on the KubeFed host cluster
  - `multicloud-demo` Deployment/Service present in all member clusters
- `istioctl` installed locally
- Permission to install Istio on each cluster

> You can enable Istio gradually (cluster by cluster).  
> The KubeFed resources (app) are independent of Istio.

---

## 1. Install Istio on each cluster

For each cluster (e.g. `aws-us-east-1` and `gcp-europe-west1`), run:

```bash
# Example for AWS cluster
istioctl install --set profile=default -y --context aws-us-east-1

# Example for GCP cluster
istioctl install --set profile=default -y --context gcp-europe-west1
```

This installs:

- `istiod` (Istio control plane)
- `istio-ingressgateway` in namespace `istio-system`
- Necessary CRDs

> In production you’ll likely want a more customized `IstioOperator` manifest.

---

## 2. Enable sidecar injection in the namespace

The base manifests already create the namespace `multicloud-demo`.  
If it does **not** have the injection label yet, add it (per cluster):

```bash
kubectl --context aws-us-east-1 label namespace multicloud-demo istio-injection=enabled --overwrite
kubectl --context gcp-europe-west1 label namespace multicloud-demo istio-injection=enabled --overwrite
```

New Pods in that namespace will get an Istio sidecar.

To update existing Pods, restart the Deployment in each cluster (or wait for a rollout):

```bash
kubectl --context aws-us-east-1 -n multicloud-demo rollout restart deploy/multicloud-demo
kubectl --context gcp-europe-west1 -n multicloud-demo rollout restart deploy/multicloud-demo
```

---

## 3. Apply Istio add-on manifests

The common Istio resources for this POC are in:

- `kubernetes/istio-addon-manifests.yaml`

They include:

- `PeerAuthentication` (mTLS STRICT for namespace `multicloud-demo`)
- `DestinationRule` for `multicloud-demo`
- `Gateway` + `VirtualService` exposing `demo.multicloud.example.com` over HTTPS

Apply them in **each cluster**:

```bash
kubectl --context aws-us-east-1 apply -f kubernetes/istio-addon-manifests.yaml
kubectl --context gcp-europe-west1 apply -f kubernetes/istio-addon-manifests.yaml
```

> You can customize the host name used by the Gateway/VirtualService by editing
> `kubernetes/istio-addon-manifests.yaml` (default: `demo.multicloud.example.com`).

---

## 4. Create the TLS Secret for the Istio Gateway

The Istio `Gateway` expects a TLS secret named `multicloud-demo-tls` in namespace `istio-system`.

You can use a **self-signed** certificate for the POC, or a real one (e.g. via cert-manager).

Example using local `tls.crt` / `tls.key` files (same for each cluster):

```bash
# AWS cluster
kubectl --context aws-us-east-1 -n istio-system create secret tls multicloud-demo-tls \
  --cert=tls.crt \
  --key=tls.key

# GCP cluster
kubectl --context gcp-europe-west1 -n istio-system create secret tls multicloud-demo-tls \
  --cert=tls.crt \
  --key=tls.key
```

If you’re using a self-signed cert, you’ll need to trust it in your browser/HTTP client as usual.

> In a more advanced setup, you’d likely use **cert-manager** with Let’s Encrypt to issue a cert for `demo.multicloud.example.com` in each cluster.

---

## 5. Ensure the Istio Ingress Gateway exposes HTTPS (LoadBalancer)

The profile `default` usually creates a `Service` named `istio-ingressgateway` in `istio-system`.  
Verify it has type `LoadBalancer` and port 443/TCP exposed.

You can patch or re-apply as needed; a minimal example is:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: istio-ingressgateway
  namespace: istio-system
spec:
  type: LoadBalancer
  selector:
    istio: ingressgateway
  ports:
    - name: https
      port: 443
      targetPort: 8443
      protocol: TCP
```

Apply per cluster if necessary:

```bash
kubectl --context aws-us-east-1 apply -f istio-ingressgateway-service.yaml
kubectl --context gcp-europe-west1 apply -f istio-ingressgateway-service.yaml
```

Once created, note the external IP / hostname in each cluster:

```bash
kubectl --context aws-us-east-1 -n istio-system get svc istio-ingressgateway
kubectl --context gcp-europe-west1 -n istio-system get svc istio-ingressgateway
```

---

## 6. DNS / host configuration

For a quick test you can:

- Use `/etc/hosts` to map `demo.multicloud.example.com` to one of the ingress IPs, or
- Set up DNS (e.g. Route 53) with records pointing to the ingress gateways

Example using `/etc/hosts` (for a quick POC):

```text
<EXTERNAL_IP_AWS> demo.multicloud.example.com
```

Then:

```bash
curl -k https://demo.multicloud.example.com/
```

You should see JSON similar to:

```json
{
  "hostname": "multicloud-demo-6548f7ffbd-2s7gh",
  "region": "us-east-1",
  "podIP": "10.42.1.23",
  "time": "2025-01-01T12:34:56Z"
}
```

If you switch `/etc/hosts` to the GCP IP, the `region` field should change to `europe-west1`.

For a full multi-region experience you can:

- Use **Route 53** (or Cloudflare, etc.) with latency/failover routing,
- Point `demo.multicloud.example.com` to both ingress gateways,
- Let DNS decide which cluster is used per client/region.

---

## 7. How mTLS works in this POC

With the Istio add-on manifests applied:

- `PeerAuthentication` in `multicloud-demo` namespace sets `mtls.mode = STRICT`  
  → all service-to-service traffic in the namespace must be **mutual TLS**.
- `DestinationRule` for `multicloud-demo` uses `tls.mode = ISTIO_MUTUAL`  
  → Istio sidecars handle certificate exchange and authenticate workloads automatically.
- `Gateway` + `VirtualService`: external HTTPS is terminated at the Istio ingress gateway, then traffic is re-encrypted as mTLS inside the mesh.

Path of a request:

1. Client → HTTPS (`demo.multicloud.example.com`) → Istio ingress gateway (cert from `multicloud-demo-tls`)
2. Istio ingress gateway → mTLS → demo app Pods in `multicloud-demo` namespace
3. Response returns via the same path, encrypted end-to-end

The **application code** doesn’t need to know anything about TLS; all encryption is handled by Istio sidecars and the gateway.

---

## 8. Cleaning up Istio add-on resources

To remove only the Istio add-on resources (keeping the base POC):

```bash
# Istio add-on (PeerAuth, DestRule, Gateway, VirtualService)
kubectl --context aws-us-east-1 delete -f kubernetes/istio-addon-manifests.yaml
kubectl --context gcp-europe-west1 delete -f kubernetes/istio-addon-manifests.yaml

# Optional: delete TLS secrets in istio-system
kubectl --context aws-us-east-1 -n istio-system delete secret multicloud-demo-tls
kubectl --context gcp-europe-west1 -n istio-system delete secret multicloud-demo-tls
```

If you want to fully remove Istio itself, use `istioctl x uninstall` or your original installation method.

---

## 9. Next steps / ideas

- Add more clusters (regions/clouds) to KubeFed and Istio
- Use **Gateway API** instead of classic Istio Gateway + VirtualService
- Integrate **cert-manager** to auto-issue certificates from Let’s Encrypt
- Add **AuthorizationPolicy** to restrict which services/namespaces can call the demo app
- Integrate with GitOps (Argo CD / Flux) to manage Istio and KubeFed config across clusters
