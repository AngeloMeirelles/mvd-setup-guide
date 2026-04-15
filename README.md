# MinimumViableDataspace (MVD) — Local Setup Guide

End-to-end guide to run the Eclipse Dataspace Components [MinimumViableDataspace](https://github.com/eclipse-edc/MinimumViableDataspace) locally on **Windows 11 + WSL2** using **kind**, **Traefik** (Gateway API), and **Bruno** for testing.

This guide documents a full walkthrough that was validated end-to-end — including the non-obvious workarounds you hit on WSL2/kind. If you follow every step, the final verification is a `200 OK` from a `Request Catalog` call returning the provider's dataset offers.

---

## 1. Prerequisites

All commands run inside **Ubuntu on WSL2** unless noted otherwise.

### Software
- **Windows 11** with WSL2 + Ubuntu distro
- **Docker Desktop for Windows** with WSL integration enabled for Ubuntu
  - Settings → Resources → WSL Integration → toggle on for your Ubuntu distro → Apply & Restart
- **Java 17** (`sudo apt install openjdk-17-jdk`)
- **kind** (Kubernetes in Docker) — https://kind.sigs.k8s.io/docs/user/quick-start/#installation
- **kubectl** — https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
- **helm** — https://helm.sh/docs/intro/install/
- **git**
- **Bruno** (API client, Windows app) — https://www.usebruno.com/

### Sanity checks

```bash
docker info            # should succeed without sudo
kind version
kubectl version --client
helm version
java -version
```

If `docker info` fails with `command could not be found`, re-check Docker Desktop's WSL Integration toggle.

### Important: repo location

Clone the MVD repo **inside the Ubuntu home directory** (`~/`), not under `/mnt/c/...`. Cloning under `/mnt/c` causes git permission issues that break builds and seed jobs.

---

## 2. Clone the repository

```bash
cd ~
git clone https://github.com/eclipse-edc/MinimumViableDataspace.git
cd MinimumViableDataspace
```

---

## 3. Create the Kubernetes cluster

```bash
kind create cluster --name mvd
kubectl cluster-info --context kind-mvd
```

Expected: control plane URL and CoreDNS URL printed.

If you get `node(s) already exist for a cluster with the name "mvd"`, either reuse it (skip to step 4) or recreate with:

```bash
kind delete cluster --name mvd && kind create cluster --name mvd
```

---

## 4. Install the Gateway API CRDs

MVD's manifests use Gateway API resources (`GatewayClass`, `Gateway`, `HTTPRoute`). Install the standard set:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

Verify:

```bash
kubectl get crd | grep gateway
```

You should see at least `gatewayclasses`, `gateways`, and `httproutes` listed.

---

## 5. Install Traefik via Helm

Traefik will be the `GatewayClass` implementation that actually routes HTTP into the cluster.

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
helm install traefik traefik/traefik --namespace traefik --create-namespace
```

Verify Traefik is running:

```bash
kubectl get pods -n traefik
```

Wait until the pod is `1/1 Running`.

---

## 6. ⚠️ CRITICAL WORKAROUND — Patch Vault manifests for kind/WSL2

The MVD manifests use `hashicorp/vault:latest` which fails to start on kind inside WSL2 with:

```
unable to set CAP_SETFCAP effective capability: Operation not permitted
```

This happens because the Vault Docker image's entrypoint tries to call `setcap cap_ipc_lock=+ep` on the Vault binary, which requires a capability the kind container doesn't expose. The fix is to set `SKIP_SETCAP=true` so the entrypoint skips that step.

**You must apply this patch before running `kubectl apply -k k8s/`**, otherwise all the `controlplane`, `identityhub`, and `issuerservice` pods will crashloop waiting for Vault.

Run this from the repo root:

```bash
for f in k8s/consumer/base/vault.yaml k8s/provider/base/vault.yaml k8s/issuer/base/vault.yaml; do
  sed -i 's|value: "0.0.0.0:8200"|value: "0.0.0.0:8200"\n            - name: SKIP_SETCAP\n              value: "true"|' "$f"
done
```

Verify all three files got the env var:

```bash
grep -A1 SKIP_SETCAP k8s/consumer/base/vault.yaml k8s/provider/base/vault.yaml k8s/issuer/base/vault.yaml
```

Each file should show `SKIP_SETCAP` with `value: "true"`.

---

## 7. Apply the MVD manifests

The `k8s/` directory is a Kustomize overlay that deploys all four namespaces (`mvd-common`, `consumer`, `provider`, `issuer`) in one shot:

```bash
kubectl apply -k k8s/
```

Watch the pods come up:

```bash
kubectl get pods -A -w
```

Expected final state (may take 2–4 minutes):

| Namespace    | Pod pattern                                  | Expected |
|--------------|----------------------------------------------|----------|
| `mvd-common` | `keycloak-*`, `postgres-*`                   | Running  |
| `consumer`   | `controlplane-*`, `identityhub-*`, `vault-*`, `postgres-*` | Running |
| `provider`   | `controlplane-*`, `dataplane-*`, `identityhub-*`, `vault-*`, `postgres-*` | Running |
| `issuer`     | `issuerservice-*`, `vault-*`, `postgres-*`   | Running  |
| Seed jobs    | `*-seed-*`, `vault-bootstrap-*`              | Completed |

Press `Ctrl+C` to exit watch mode.

---

## 8. Configure `/etc/hosts` for local hostnames

The MVD `HTTPRoute` resources use hostnames like `cp.consumer.localhost`. Ubuntu does **not** automatically resolve `*.localhost` to `127.0.0.1`, so we add explicit entries.

> **Pitfall:** Do not paste a long `printf` one-liner into a terminal that auto-indents pasted text — it will wrap and corrupt `/etc/hosts`. Use short `echo | sudo tee -a` commands instead.

```bash
echo "127.0.0.1 cp.consumer.localhost"    | sudo tee -a /etc/hosts > /dev/null
echo "127.0.0.1 ih.consumer.localhost"    | sudo tee -a /etc/hosts > /dev/null
echo "127.0.0.1 vault.consumer.localhost" | sudo tee -a /etc/hosts > /dev/null
echo "127.0.0.1 cp.provider.localhost"    | sudo tee -a /etc/hosts > /dev/null
echo "127.0.0.1 ih.provider.localhost"    | sudo tee -a /etc/hosts > /dev/null
echo "127.0.0.1 dp.provider.localhost"    | sudo tee -a /etc/hosts > /dev/null
echo "127.0.0.1 vault.provider.localhost" | sudo tee -a /etc/hosts > /dev/null
echo "127.0.0.1 issuer.localhost"         | sudo tee -a /etc/hosts > /dev/null
echo "127.0.0.1 keycloak.localhost"       | sudo tee -a /etc/hosts > /dev/null
```

Verify:

```bash
getent hosts cp.consumer.localhost
```

Must return `127.0.0.1 cp.consumer.localhost`.

---

## 9. Port-forward Traefik on port 80

The Bruno collection uses URLs without explicit ports (`http://cp.consumer.localhost`), which means port 80. Binding port 80 requires root, and `sudo` by default uses root's empty kubeconfig — so you must pass `KUBECONFIG` explicitly.

Open a **dedicated terminal window** and run:

```bash
sudo KUBECONFIG=$HOME/.kube/config kubectl port-forward -n traefik svc/traefik 80:80
```

Expected output:

```
Forwarding from 127.0.0.1:80 -> 80
Forwarding from [::1]:80 -> 80
```

**Leave this window open.** Closing it tears down the tunnel.

---

## 10. Smoke-test the routing

In a **second terminal**:

```bash
curl -v http://cp.consumer.localhost/api/mgmt/v4beta/catalog/request 2>&1 | head -20
```

Any HTTP response from the **Jetty** server (405, 404, 401, 400, etc.) with an HTML body mentioning `Error N` means success — Traefik is routing correctly and the EDC controlplane is up.

If you instead get the plaintext `404 page not found` — that's Traefik saying it couldn't match any `HTTPRoute`. Check that `/etc/hosts` actually has the entry and the port-forward is on port 80.

---

## 11. Open the Bruno collection and run Request Catalog

1. From WSL, open the `Requests` folder in Windows Explorer:
   ```bash
   cd ~/MinimumViableDataspace/Requests && explorer.exe .
   ```
2. In the opened Explorer window, copy the address bar value (something like `\\wsl.localhost\Ubuntu\home\<user>\MinimumViableDataspace\Requests`).
3. In Bruno, click **Open Collection**, paste that path, and confirm.
4. In the collection sidebar, expand **ControlPlane Management** → click **Request Catalog**.
5. Environment dropdown (top right) can stay on **No Environment** — the collection defines its variables in `collection.bru` (`CONSUMER_CP`, `PROVIDER_DSP`, `PROVIDER_ID`, etc.).
6. Click **Send**.

### Expected result — the "hello world" of the dataspace

Status **200 OK**, with a JSON body like:

```json
{
  "@type": "Catalog",
  "dataset": [
    {
      "@id": "asset-1",
      "hasPolicy": [...],
      "distribution": [...],
      "description": "This asset requires Membership to view and Manufacturer (part_types=non_critical) to negotiate."
    },
    {
      "@id": "asset-2",
      "description": "This asset requires Membership to view and Manufacturer (part_types=all) to negotiate."
    }
  ],
  "participantId": "did:web:identityhub.provider.svc.cluster.local%3A7083:provider"
}
```

🎉 **If you see that, the whole stack works**: Bruno → Traefik → consumer controlplane → (DSP, internal) → provider controlplane → catalog back to Bruno. IdentityHub, Vault, Keycloak, Postgres are all exercised by this single round-trip.

---

## 12. Next requests to explore

Inside **ControlPlane Management**:

1. **Initiate negotiation** — pick one of the catalog assets and start a contract negotiation.
2. **Get Contract Negotiations** — poll until the state becomes `FINALIZED`.
3. **Initiate Transfer** — once finalized, start a data transfer.
4. **Get cached EDRs** → **Download Data from Public API** — consume the actual data.

---

## Troubleshooting

### Vault pods in `CrashLoopBackOff` with `CAP_SETFCAP` error

You forgot step 6. Apply the `sed` patch, re-run `kubectl apply -k k8s/`, then:

```bash
kubectl rollout restart deployment/controlplane deployment/identityhub -n consumer
kubectl rollout restart deployment/controlplane deployment/identityhub -n provider
kubectl rollout restart deployment/issuerservice -n issuer
```

The rollout restarts avoid long exponential backoff on the dependent pods.

### `kubectl port-forward` on port 80 fails with permission denied

You need `sudo`, and you also need to pass `KUBECONFIG` explicitly because root has a different HOME:

```bash
sudo KUBECONFIG=$HOME/.kube/config kubectl port-forward -n traefik svc/traefik 80:80
```

### `curl http://cp.consumer.localhost/...` returns `404 page not found` (plaintext)

That's Traefik failing to match a route. Two common causes:
1. The hostname isn't in `/etc/hosts` — `getent hosts cp.consumer.localhost` should return `127.0.0.1`.
2. Port-forward is on 8080 instead of 80 (and you're therefore hitting whatever listens on Windows port 80, or getting a refusal).

### `docker info` fails with `command could not be found`

Docker Desktop WSL integration isn't enabled for your distro. Fix in Docker Desktop → Settings → Resources → WSL Integration.

### Seed jobs stuck in `Init:0/1`

Their init containers wait for the corresponding service (`controlplane`, `identityhub`, `issuerservice`) to become ready. If those pods are crashlooping (usually because Vault is down), fix Vault first and the seeds will complete automatically.

### Bruno can't see the WSL path

Use `\\wsl.localhost\Ubuntu\home\<user>\...` in the Bruno Open Collection dialog. If that fails, copy the `Requests` folder to a Windows path (e.g. `C:\Users\<user>\mvd-requests\`) and open from there — the collection works regardless of location.

---

## Known quirks (not blockers)

- In `k8s/issuer/base/vault.yaml`, the `HTTPRoute` hostname for the issuer's Vault is `vault.provider.localhost` (same as the provider's Vault). This looks like an upstream typo — it doesn't affect the smoke test but you'd need to fix it if you want to hit the issuer's Vault via Traefik.

---

## Reference

- Upstream repo: https://github.com/eclipse-edc/MinimumViableDataspace
- Eclipse EDC: https://github.com/eclipse-edc
- Gateway API: https://gateway-api.sigs.k8s.io/
- Traefik Helm chart: https://github.com/traefik/traefik-helm-chart
- Bruno: https://www.usebruno.com/
