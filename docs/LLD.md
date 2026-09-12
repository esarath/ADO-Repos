# Low-Level Design — Command Reference

Every command actually run to build the `dev` → `stg` → `prod` pipeline for
`pipeline-dotNet`, grouped by concern, with a one-line explanation each.
Read [HLD.md](./HLD.md) first for the "why"; use this as the copy-paste
reference when repeating the pattern for a new project or environment.

Placeholders: `<NS>` = target namespace (`ado-pipeline` / `pipeline-stg` /
`pipeline-prod`), `<ENV>` = short env name (`dev` / `stg` / `prod`).

---

## 1. Self-hosted ADO agent (one-time, per agent host)

Needed because the OCP API and the local registry are LAN-only —
Microsoft-hosted agents can't reach either.

```bash
# Install runtime deps for the agent (libicu etc.)
sudo dnf install -y git-subtree   # unrelated one-off, only needed for
                                   # git subtree add on this host

mkdir -p ~/ado-agent && cd ~/ado-agent
curl -sL -o agent.tar.gz \
  https://download.agent.dev.azure.com/agent/5.279.0/pipelines-agent-linux-x64-5.279.0.tar.gz
tar xzf agent.tar.gz
sudo ./bin/installdependencies.sh   # installs missing OS packages (e.g. lttng-ust)

# Interactive — do NOT pass the PAT as a CLI arg, let it prompt (hidden input)
./config.sh --url https://dev.azure.com/mydevlabs0 --auth pat \
  --pool lab-ocp-agents --agent centos-lab-agent --acceptTeeEula

# Register + start as a systemd service so it survives reboots
sudo ./svc.sh install centos
sudo ./svc.sh start
sudo ./svc.sh status   # expect: active (running)
```

**Portal prerequisites** (must exist before `config.sh` runs):
- Org Settings → **Pipelines → Agent pools → Add pool** (self-hosted) →
  name `lab-ocp-agents`.
- Your icon → **Personal access tokens → New Token** → scope
  **Agent Pools (Read & manage)**.

---

## 2. Local registry auth (one-time gotcha, applies to every env)

```bash
# WRONG file — this is a stale copy, not what the registry reads:
#   ~/users.htpasswd
# RIGHT file — found via the registry's Quadlet unit
#   (/etc/containers/systemd/registry.container →
#    Volume=/opt/registry/auth:/auth, REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd)
sudo htpasswd -B /opt/registry/auth/htpasswd developer   # interactive password prompt
sudo systemctl restart registry.service

podman login svc-infra.ocp.local:5000   # username: developer
```

TLS trust for `podman` against this registry's self-signed/internal CA was
already present at `/etc/containers/certs.d/svc-infra.ocp.local:5000/ca.crt`
— nothing to do there. (`docker` does **not** share this trust store; the
pipeline uses `podman` for build/push specifically to avoid needing to
configure `/etc/docker/certs.d/` separately.)

---

## 3. Per-environment cluster setup (repeat for dev / stg / prod)

### 3.1 Namespace
```bash
oc get namespace <NS> || oc create namespace <NS>
```

### 3.2 Scoped service account + RBAC
```bash
oc create serviceaccount ado-deployer -n <NS>
oc policy add-role-to-user edit -z ado-deployer -n <NS>
```
Binds the `edit` ClusterRole via a **namespace-scoped `RoleBinding`**
(never `ClusterRoleBinding`) — this SA can only manage workloads inside
`<NS>`.

> **Production note:** for `pipeline-prod`, run this command yourself,
> deliberately, one namespace at a time — do not script it in a loop
> across environments.

### 3.3 Long-lived token secret
OCP 4.20+ no longer auto-generates a token secret for new service
accounts, so create one explicitly:
```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ado-deployer-token
  namespace: <NS>
  annotations:
    kubernetes.io/service-account.name: ado-deployer
type: kubernetes.io/service-account-token
EOF
```

### 3.4 Build a scoped kubeconfig from the token
```bash
NS=<NS>
TOKEN=$(oc get secret ado-deployer-token -n "$NS" -o jsonpath='{.data.token}' | base64 -d)
oc get secret ado-deployer-token -n "$NS" -o jsonpath='{.data.ca\.crt}' | base64 -d \
  > $HOME/POCs/pipeline/ado-secrets/ado-deployer-${NS}-ca.crt

KCFG=$HOME/POCs/pipeline/ado-secrets/ado-deployer-${NS}.kubeconfig
KUBECONFIG="$KCFG" oc config set-cluster lab-ocp-local \
  --server=https://api.lab.ocp.local:6443 \
  --certificate-authority=$HOME/POCs/pipeline/ado-secrets/ado-deployer-${NS}-ca.crt \
  --embed-certs=true
KUBECONFIG="$KCFG" oc config set-credentials ado-deployer --token="$TOKEN"
KUBECONFIG="$KCFG" oc config set-context ado-deployer-ctx \
  --cluster=lab-ocp-local --user=ado-deployer --namespace="$NS"
KUBECONFIG="$KCFG" oc config use-context ado-deployer-ctx
chmod 600 "$KCFG"

KUBECONFIG="$KCFG" oc whoami   # expect: system:serviceaccount:<NS>:ado-deployer
```

> **Gotcha:** use `$HOME`, not `~`, inside `--flag=value` arguments — bash
> only tilde-expands a bare word or a plain `VAR=value` assignment, not
> `~` following `=` inside a long-option flag (`--certificate-authority`
> isn't a valid shell identifier, so it's never treated as an assignment).
> `~/path` silently fails to expand there and gets passed to `oc` literally.

### 3.5 ADO Environment + approval gate (portal; skip approval for dev)
1. **Pipelines → Environments → New environment** → name `<ENV>` →
   Resource: None → Create.
2. `<ENV>` → **⋮ → Approvals and checks → + → Approvals** → add
   approver(s) → Create. (Use a stricter/different approver set for
   `prod` than for `stg`.)

### 3.6 Kubernetes service connection (portal)
1. **Project Settings → Pipelines → Service connections → New service
   connection → Kubernetes → Kubeconfig**.
2. Paste `cat $HOME/POCs/pipeline/ado-secrets/ado-deployer-<NS>.kubeconfig`.
3. Cluster context: `ado-deployer-ctx`. Namespace: `<NS>`.
4. **Uncheck "Verify connection"** — ADO's cloud backend can't resolve
   `api.lab.ocp.local` or route to `192.168.29.0/24`; verification always
   fails here even though the connection works at run time via the
   self-hosted agent.
5. Name: `ocp-lab-<NS>` (dev used `ocp-lab-ado-pipeline-dev`) → Save.
6. Leave "Grant access permission to all pipelines" **unchecked** —
   authorize per-pipeline on first use instead.

### 3.7 Verify after a pipeline run
```bash
oc get pods -n <NS>
```
Always pass `-n` explicitly — `oc get pods`/`oc get all` with no
namespace flag uses whatever your **current shell's** kubeconfig context
happens to default to, which changes every time you run
`oc config use-context` while building a different environment's
kubeconfig (easy to misread which environment you're actually looking at).

---

## 4. Container image (`pipeline-dotNet/Dockerfile`)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish pipelines-dotnet-core.csproj -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS runtime
WORKDIR /app
COPY --from=build /app/publish .
RUN chmod -R g=u /app
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV HOME=/tmp
ENTRYPOINT ["dotnet", "pipelines-dotnet-core.dll"]
```

- **`chmod -R g=u /app` + `ENV HOME=/tmp`**: OpenShift's restricted SCC
  runs containers as an arbitrary, unpredictable non-root UID (not the
  UID baked into the image). Without this, ASP.NET Core's DataProtection
  key storage (`~/.aspnet/DataProtection-Keys`) can't be written and
  antiforgery/session features can throw. Verified locally:
  ```bash
  podman build -t <repo>/pipeline-dotnet:test -f pipeline-dotNet/Dockerfile pipeline-dotNet
  podman run -d --rm -p 18080:8080 --user 1001 <repo>/pipeline-dotnet:test
  curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:18080/   # HTTP 200
  ```

## 5. Kubernetes manifests (`pipeline-dotNet/k8s/`)

`deployment.yaml` and `service.yaml` intentionally have **no
`metadata.namespace` field**. If they did, `kubectl apply -n <NS>` would
fail with a namespace-mismatch error the moment the same manifest is
applied to a different namespace than the one hardcoded in the file — the
namespace is supplied per-stage instead, via the pipeline task (Section 6).

## 6. Pipeline (`azure-pipelines-1.yml`) — stage pattern

```yaml
variables:
  imageRepo: 'svc-infra.ocp.local:5000/ado-pipeline/pipeline-dotnet'
  imageTag: '$(Build.BuildId)'   # immutable per-run tag, promoted as-is

stages:
- stage: Build
  jobs:
  - job: Build
    pool: { name: 'lab-ocp-agents' }
    steps:
    - task: UseDotNet@2
      inputs: { packageType: 'sdk', version: '6.0.x' }
    - task: DotNetCoreCLI@2
      inputs: { command: 'restore', projects: '$(projectPath)' }
    - task: DotNetCoreCLI@2
      inputs: { command: 'build', projects: '$(projectPath)', arguments: '--configuration $(buildConfiguration) --no-restore' }
    - script: |
        set -e
        podman build -t $(imageRepo):$(imageTag) -t $(imageRepo):latest -f pipeline-dotNet/Dockerfile pipeline-dotNet
        podman push $(imageRepo):$(imageTag)
        podman push $(imageRepo):latest
    - publish: pipeline-dotNet/k8s
      artifact: manifests

- stage: Deploy            # dev — no approval
  dependsOn: Build
  jobs:
  - deployment: DeployToOcp
    environment: 'dev'
    pool: { name: 'lab-ocp-agents' }
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@1
            inputs:
              connectionType: 'kubernetesServiceConnection'
              kubernetesServiceConnection: 'ocp-lab-ado-pipeline-dev'
              namespace: 'ado-pipeline'
              manifests: '$(Pipeline.Workspace)/manifests/*.yaml'
              containers: '$(imageRepo):$(imageTag)'

# DeployStg and DeployProd repeat the Deploy shape exactly:
#   dependsOn: <previous stage id>   (Deploy, then DeployStg)
#   environment: 'stg' / 'prod'      (this is what pauses for approval)
#   kubernetesServiceConnection: 'ocp-lab-pipeline-stg' / 'ocp-lab-pipeline-prod'
#   namespace: 'pipeline-stg' / 'pipeline-prod'
# — same $(imageTag), no rebuild.
```

**Common mistakes made (and fixed) while building this** — worth
remembering when extending the pattern:
- `dependsOn` must reference a stage's short `stage:` id (e.g. `Deploy`),
  not its `displayName` or an invented name like `Dev`.
- `kubernetesServiceConnection` must be the exact service connection name
  created in Section 3.6 — a leftover template placeholder like
  `ocp-lab-<namespace>` produces "service connection ... could not be
  found" at pipeline-validation time, not a runtime error.
- Editing the pipeline YAML in the ADO portal's editor **replaces the
  whole file** if you paste over it rather than inserting — this once
  deleted the `Build` and dev `Deploy` stages entirely and left a stray
  `---` YAML document separator (invalid mid-file). Prefer editing in git
  and pushing, or use the portal's editor to insert at a specific line
  rather than pasting a full-file replacement.

---

## 7. Security best practices applied

See [HLD.md §5](./HLD.md#5-key-design-decisions-best-practices) for the
full list (least privilege, no shared credentials, no committed secrets,
immutable image promotion, namespace-agnostic manifests, arbitrary-UID
hardening, deliberate prod changes, per-pipeline connection
authorization). Token rotation, if ever needed:
```bash
oc delete secret ado-deployer-token -n <NS>
# re-run 3.3 and 3.4, then update the ADO service connection with the new kubeconfig
```
