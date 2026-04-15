# Instructions for Claude Code

This repository exists so the user (Angelo) can clone it on any machine and have Claude Code guide them through setting up the Eclipse EDC **MinimumViableDataspace** locally for study. The full step-by-step guide is in `README.md`.

## Who the user is and what they want

Angelo is learning the EDC dataspace stack hands-on. This is a **local learning sandbox**, not production. He wants to reproduce the exact setup that was validated end-to-end on his primary machine, then explore the full dataspace flow (catalog → negotiation → transfer → consume) via Bruno.

## How to collaborate with him

- **Guide one command at a time.** Wait for his output/confirmation before moving to the next step. Do not dump a 15-command script and ask him to run them all.
- **Write in Portuguese (pt-BR)** when talking to him. The working language of the original session was Portuguese.
- **Short explanations before each command.** He wants to understand what each step does, not just copy-paste.
- **Assume commands run inside Ubuntu/WSL2** unless explicitly stated otherwise. Never suggest moving the repo to `/mnt/c/...` — that causes git permission issues that break MVD builds.
- **If Docker permission issues appear**, the fix is adding the user to the `docker` group, never `sudo docker`.
- **Don't skip the workarounds in README.md §6, §8, §9.** They are not optional on kind/WSL2.

## Environment assumptions

- Windows 11 + WSL2 + Ubuntu
- Docker Desktop with WSL integration enabled
- Java 17, kind, kubectl, helm, git, Bruno all installed beforehand

If any of these are missing, install them first (guide him through apt installs or pointer to official docs) before attempting the MVD setup.

## The three non-obvious workarounds — memorize these

These are the parts that will waste hours if forgotten. The full explanations are in `README.md`; this is the quick reference.

### 1. Vault `SKIP_SETCAP=true` patch (README §6)

`hashicorp/vault:latest` in `k8s/{consumer,provider,issuer}/base/vault.yaml` fails on kind inside WSL2 with `unable to set CAP_SETFCAP effective capability: Operation not permitted`. Patch all three files to add `SKIP_SETCAP=true` as an env var before `kubectl apply -k k8s/`:

```bash
for f in k8s/consumer/base/vault.yaml k8s/provider/base/vault.yaml k8s/issuer/base/vault.yaml; do
  sed -i 's|value: "0.0.0.0:8200"|value: "0.0.0.0:8200"\n            - name: SKIP_SETCAP\n              value: "true"|' "$f"
done
```

If he has already applied and the Vault pods are crashlooping, patching and re-applying is enough, but he'll also need `kubectl rollout restart` on `controlplane`/`identityhub`/`issuerservice` deployments so they retry immediately instead of waiting on a long exponential backoff.

### 2. `/etc/hosts` entries (README §8)

`*.localhost` does **not** auto-resolve on Ubuntu. The MVD `HTTPRoute` hostnames must be manually added. **Never use long `printf` one-liners with `\n` separators** to add them — the user's terminal auto-indents pasted text and wraps long lines, which corrupted `/etc/hosts` twice in the original session. Use short `echo "..." | sudo tee -a /etc/hosts` commands, one per line. The full set is in README §8.

### 3. `sudo` port-forward on port 80 with explicit `KUBECONFIG` (README §9)

The Bruno collection uses port 80 implicitly. Binding port 80 requires root, **but** `sudo kubectl` with no extra args uses root's empty kubeconfig and fails. The working command is:

```bash
sudo KUBECONFIG=$HOME/.kube/config kubectl port-forward -n traefik svc/traefik 80:80
```

Has to run in a dedicated terminal that stays open.

## Validating success

The terminal-side smoke test is:

```bash
curl -v http://cp.consumer.localhost/api/mgmt/v4beta/catalog/request 2>&1 | head -20
```

If the body is Jetty's HTML error page (`<title>Error 404 Not Found</title>`), the routing works — 404 is from the EDC app, not from Traefik. If it's the plaintext `404 page not found`, Traefik didn't match a route.

The end-to-end test is running **Request Catalog** in the Bruno collection (`ControlPlane Management` folder). Success = 200 OK with a JSON catalog containing `asset-1` and `asset-2` with policies on `ManufacturerCredential.part_types`.

## Pointers inside the upstream repo

Inside `~/MinimumViableDataspace`:

- `k8s/` — the Kustomize tree applied with `kubectl apply -k k8s/`. Four participants: `common`, `consumer`, `provider`, `issuer`.
- `k8s/*/base/vault.yaml` — the three files that need the `SKIP_SETCAP` patch.
- `Requests/` — the Bruno collection. Open it in Bruno via `explorer.exe .` from this folder, then copy the `\\wsl.localhost\...` path into Bruno's Open Collection dialog.
- `Requests/collection.bru` — defines all the env variables (`CONSUMER_CP`, `PROVIDER_DSP`, `PROVIDER_ID`, etc.) and the `X-Api-Key: password` auth. No separate environment file is needed.
- `Requests/ControlPlane Management/` — the main flow: Request Catalog → Initiate negotiation → Get Contract Negotiations → Initiate Transfer → Get cached EDRs → Download Data from Public API.

## Reference URLs to hit from Bruno / curl

Resolved via `/etc/hosts` + port-forward on port 80:

| Hostname                        | What                          |
|---------------------------------|-------------------------------|
| `cp.consumer.localhost`         | Consumer controlplane         |
| `ih.consumer.localhost`         | Consumer IdentityHub          |
| `vault.consumer.localhost`      | Consumer Vault                |
| `cp.provider.localhost`         | Provider controlplane         |
| `ih.provider.localhost`         | Provider IdentityHub          |
| `dp.provider.localhost`         | Provider dataplane (public API) |
| `vault.provider.localhost`      | Provider Vault                |
| `issuer.localhost`              | Issuer service                |
| `keycloak.localhost`            | Keycloak (mvd-common)         |

Note: `PROVIDER_DSP` in `collection.bru` is `http://controlplane.provider.svc.cluster.local:8082/api/dsp/2025-1` — that's an **in-cluster** DNS name, used by the consumer controlplane to reach the provider **inside** Kubernetes. It is not supposed to resolve from outside the cluster.
