# Deploying `prometheus/prometheus-mcp` on your k3s cluster

## 0. Your Prometheus Service

Confirmed from your cluster:

- Helm release name: `prometheus-stack`
- Service: `prometheus-stack-kube-prom-prometheus`
- Namespace: `monitoring`
- Port: `9090` (the `8080` on that Service is Prometheus's own self-scrape metrics port, not used here)

In-cluster URL:

```
http://prometheus-stack-kube-prom-prometheus.monitoring.svc.cluster.local:9090
```

This value is already filled in below.

## 1. Create a namespace (optional but recommended)

```bash
kubectl create namespace mcp-servers
```

Keeping MCP servers in their own namespace makes NetworkPolicy scoping (step 3) simpler,
consistent with how you've hardened the kubernetes-mcp-server.

## 2. Install via Helm

```bash
helm install prometheus-mcp-server oci://ghcr.io/tjhop/charts/prometheus-mcp-server \
  --version <latest-version> \
  --namespace mcp-servers \
  --set prometheus.url=http://prometheus-stack-kube-prom-prometheus.monitoring.svc.cluster.local:9090 \
  --set mcp.transport=http \
  --set service.type=ClusterIP \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.labels.release=prometheus-stack
```

Check the latest chart version in the [Releases page](https://github.com/prometheus/prometheus-mcp/releases).

The `serviceMonitor.labels.release=prometheus-stack` line assumes your kube-prometheus-stack's
`Prometheus` CR selects ServiceMonitors via a `release: prometheus-stack` label (the common default
for that chart). Confirm before relying on it:

```bash
kubectl get prometheus -n monitoring -o yaml | grep -A5 serviceMonitorSelector
```

If the selector uses a different label/value, adjust `serviceMonitor.labels` to match — or drop
those two `--set` flags entirely if you don't want the MCP server's own metrics scraped for now.

Notes on defaults (no extra flags needed for these):
- `mcp.transport` already defaults to `http`, listening on container port `8080`.
- `tsdbAdmin.enabled` defaults to `false` — the destructive TSDB admin tools
  (`snapshot`, `delete_series`, `clean_tombstones`) stay disabled unless you opt in.
- `mcp.tools` defaults to `["all"]` (all non-destructive tools). To trim the tool
  list for a smaller context window, set e.g. `--set mcp.tools[0]=core`.
- Pod/container security contexts are already restricted-PSS compliant.
- `serviceAccount.automountServiceAccountToken` defaults to `false` (the server
  doesn't talk to the Kubernetes API, only to Prometheus).

If your kube-prometheus-stack is deployed with the Prometheus Operator and you want
the MCP server's own metrics scraped into the same stack:

```bash
--set serviceMonitor.enabled=true \
--set serviceMonitor.labels.release=<your-kube-prometheus-stack-release-name>
```

(The `release` label must match whatever your `kube-prometheus-stack`'s
`serviceMonitorSelector` expects — check with
`kubectl get prometheus -n monitoring -o yaml | grep -A3 serviceMonitorSelector`.)

## 3. Lock down traffic with a NetworkPolicy

Mirroring the RBAC/NetworkPolicy hardening you did for the kubernetes-mcp-server:
restrict the MCP pod so it can only reach Prometheus on egress, and only accept
ingress from whatever namespace/pod your MCP client runs in (or none, if you're
only ever accessing it via `kubectl port-forward`, since port-forward bypasses
Services/NetworkPolicies at the API-server proxy level — but it's still good
practice for defense-in-depth once you add other pods that talk to it).

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: prometheus-mcp-server-netpol
  namespace: mcp-servers
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: prometheus-mcp-server
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        # replace with the actual client namespace/pod selector once you have one;
        # leave empty (no `from`) to block all Service-based ingress and rely on
        # kubectl port-forward only
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: <your-mcp-client-namespace>
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # Prometheus itself (prometheus-stack-kube-prom-prometheus.monitoring:9090)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 9090
    # DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

Apply it:

```bash
kubectl apply -f prometheus-mcp-server-netpol.yaml
```

## 4. Access

**In-cluster (HTTP):**

```
http://prometheus-mcp-server.mcp-servers.svc.cluster.local:8080
```

Point any in-cluster MCP client (streamable HTTP transport) at that URL.

**From outside, via port-forward:**

```bash
kubectl port-forward -n mcp-servers svc/prometheus-mcp-server 8080:8080
```

Then point your MCP client at `http://localhost:8080`.

## 5. Verify

```bash
kubectl get pods -n mcp-servers
kubectl logs -n mcp-servers deploy/prometheus-mcp-server
curl -s http://localhost:8080/metrics | head   # after port-forwarding
```

A healthy pod will expose `prom_mcp_server_ready 1` in its own `/metrics` output.