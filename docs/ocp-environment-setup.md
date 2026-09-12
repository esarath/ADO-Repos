# OCP Environment Setup Runbook — Multi-Stage Deploy (dev → stg → prod)

This is the step-by-step procedure used to wire an Azure DevOps pipeline to
deploy `pipeline-dotNet` (and future projects in this repo) into namespaces
on the `lab.ocp.local` OpenShift cluster, with manual approval gates between
environments.

**`dev` has already been completed end-to-end and is used below as the
worked example.** Repeat the same steps for `stg` and `prod`, substituting
the values in the table.

## Environment naming reference

| Env  | Namespace       | Service Account | Kubeconfig file (on svc-infra)                  | ADO Service Connection      | ADO Environment | Approval required |
|------|-----------------|------------------|--------------------------------------------------|------------------------------|------------------|--------------------|
| dev  | `ado-pipeline`  | `ado-deployer`   | `~/scratchpad/ado-deployer.kubeconfig` *(see note)* | `ocp-lab-ado-pipeline-dev`   | `dev`            | No                 |
| stg  | `pipeline-stg`  | `ado-deployer`   | `~/POCs/pipeline/ado-secrets/ado-deployer-pipeline-stg.kubeconfig`  | `ocp-lab-pipeline-stg`       | `stg`            | Yes                |
| prod | `pipeline-prod` | `ado-deployer`   | `~/POCs/pipeline/ado-secrets/ado-deployer-pipeline-prod.kubeconfig` | `ocp-lab-pipeline-prod`      | `prod`           | Yes (stricter)     |

> Note: the `dev` kubeconfig was originally written to a session-scoped
> scratchpad path that no longer exists. If you need to regenerate it,
> follow the same commands in Step 2 below with `NS=ado-pipeline`.

All environments share:
- Cluster API: `https://api.lab.ocp.local:6443`
- Container registry: `svc-infra.ocp.local:5000` (TLS + htpasswd auth; real
  auth file is `/opt/registry/auth/htpasswd` on svc-infra — **not**
  `~/users.htpasswd`, which is a stale copy)
- Self-hosted ADO agent pool: `lab-ocp-agents` (agent `centos-lab-agent`,
  runs on `svc-infra` as user `centos`) — required because the OCP API and
  the local registry are only reachable inside the home LAN; Microsoft-hosted
  agents cannot reach either.

---

## Step 1 — Confirm/create the namespace on OCP

```bash
oc get namespace <namespace>          # e.g. pipeline-stg
# if missing:
oc create namespace <namespace>
```

## Step 2 — Create a scoped service account + RBAC + token secret

Run on `svc-infra` (or wherever you have `system:admin`/cluster-admin
`oc` access):

```bash
NS=<namespace>   # e.g. pipeline-stg

oc create serviceaccount ado-deployer -n "$NS"
oc policy add-role-to-user edit -z ado-deployer -n "$NS"

cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ado-deployer-token
  namespace: $NS
  annotations:
    kubernetes.io/service-account.name: ado-deployer
type: kubernetes.io/service-account-token
EOF
```

The `edit` ClusterRole is bound only within `$NS` (via `RoleBinding`, not
`ClusterRoleBinding`), so this service account can manage workloads in that
one namespace only — it cannot touch other namespaces or cluster-scoped
resources.

**Production note:** for `pipeline-prod`, treat this as a protected action —
run it deliberately yourself rather than scripting it unattended, and
double check `NS` before hitting enter.

## Step 3 — Build a scoped kubeconfig from the token

```bash
NS=<namespace>
mkdir -p ~/POCs/pipeline/ado-secrets && chmod 700 ~/POCs/pipeline/ado-secrets

TOKEN=$(oc get secret ado-deployer-token -n "$NS" -o jsonpath='{.data.token}' | base64 -d)
oc get secret ado-deployer-token -n "$NS" -o jsonpath='{.data.ca\.crt}' | base64 -d > ~/POCs/pipeline/ado-secrets/ado-deployer-${NS}-ca.crt

KCFG=~/POCs/pipeline/ado-secrets/ado-deployer-${NS}.kubeconfig
KUBECONFIG="$KCFG" oc config set-cluster lab-ocp-local \
  --server=https://api.lab.ocp.local:6443 \
  --certificate-authority=~/POCs/pipeline/ado-secrets/ado-deployer-${NS}-ca.crt \
  --embed-certs=true

KUBECONFIG="$KCFG" oc config set-credentials ado-deployer --token="$TOKEN"

KUBECONFIG="$KCFG" oc config set-context ado-deployer-ctx \
  --cluster=lab-ocp-local --user=ado-deployer --namespace="$NS"

KUBECONFIG="$KCFG" oc config use-context ado-deployer-ctx
chmod 600 "$KCFG"

# sanity check — should print system:serviceaccount:<NS>:ado-deployer
KUBECONFIG="$KCFG" oc whoami
```

A ready-to-run version of this loop (for stg + prod together) is at
`~/POCs/pipeline/ado-secrets/build-kubeconfigs.sh` on svc-infra.

## Step 4 — Create the ADO Environment with approval gates (portal)

1. **Pipelines → Environments → New environment** → name it `stg` (or
   `prod`) → **Resource: None** → **Create**.
2. Open the environment → **⋮ (top right) → Approvals and checks → +
   → Approvals**.
3. Add the required approver(s). For `prod`, consider requiring more than
   one approver, or a different approver than `stg`, for a real change gate.
4. **Create.**

`dev` intentionally has **no** environment-level approval — deployments to
it run automatically after a successful Build.

## Step 5 — Create the Kubernetes service connection (portal)

1. **Project Settings → Pipelines → Service connections → New service
   connection → Kubernetes**.
2. Authentication method: **Kubeconfig**.
3. Paste the full contents of the kubeconfig file from Step 3
   (`cat ~/POCs/pipeline/ado-secrets/ado-deployer-<namespace>.kubeconfig`).
4. Cluster context: `ado-deployer-ctx`. Namespace: `<namespace>`.
5. **Uncheck "Verify connection"** — Azure DevOps' cloud backend cannot
   resolve `api.lab.ocp.local` or reach the `192.168.29.0/24` range, so
   verification always fails even though the connection works fine at
   pipeline-run time (the self-hosted agent, inside the LAN, can reach it).
6. Name it per the naming table above (e.g. `ocp-lab-pipeline-stg`) →
   **Save**.

## Step 6 — Extend the pipeline YAML

Each new environment is one more `stage` in `azure-pipelines-1.yml`,
`dependsOn` the previous one, referencing its `environment:` (which carries
the approval gate) and its service connection. The **same image tag**
built once in the `Build` stage is promoted through every stage — nothing
gets rebuilt for stg/prod.

```yaml
- stage: Deploy<Env>              # e.g. DeployStg / DeployProd
  displayName: 'Deploy to <namespace> (<env>)'
  dependsOn: <PreviousStageName>  # e.g. Deploy / DeployStg
  condition: succeeded()
  jobs:
  - deployment: DeployToOcp
    environment: '<env>'          # e.g. stg / prod — this is what pauses
                                   # for approval per Step 4
    pool:
      name: 'lab-ocp-agents'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@1
            inputs:
              action: 'deploy'
              connectionType: 'kubernetesServiceConnection'
              kubernetesServiceConnection: 'ocp-lab-<namespace>'
              namespace: '<namespace>'
              manifests: '$(Pipeline.Workspace)/manifests/*.yaml'
              containers: '$(imageRepo):$(imageTag)'
```

The manifests under `pipeline-dotNet/k8s/` are namespace-agnostic (no
hardcoded `namespace:` field) so the same files apply cleanly to
`ado-pipeline`, `pipeline-stg`, and `pipeline-prod` — the target namespace
is supplied per-stage via the task's `namespace` input.

## Step 7 — Run and approve

1. Push to `main` → pipeline triggers → **Build** runs, pushes the image →
   **Deploy** (dev) runs automatically.
2. **DeployStg** starts and immediately pauses — approve it under
   **Checks** on that stage in the run view (or via the environment's
   pending-approval notification).
3. Verify: `oc get pods -n pipeline-stg`.
4. **DeployProd** pauses the same way — approve, then verify:
   `oc get pods -n pipeline-prod`.

---

## Security best practices applied here

- **Least privilege, per-namespace scope.** Each environment gets its own
  `ado-deployer` service account bound to `edit` via a `RoleBinding` scoped
  to that one namespace (`oc policy add-role-to-user ... -n <namespace>`),
  never a `ClusterRoleBinding`. The stg service account cannot touch
  `pipeline-prod` or any other namespace, and vice versa.
- **No shared credentials across environments.** dev, stg, and prod each
  have their own service account, token, kubeconfig, and ADO service
  connection — compromising one does not expose the others.
- **Credentials never committed to git.** This repo's `.gitignore` blocks
  `*.kubeconfig`, `*-ca.crt`, `*.token`, `*.pem`, `*.key`, and
  `users.htpasswd`. Kubeconfigs live only in `~/POCs/pipeline/ado-secrets` (`chmod 700`
  dir, `chmod 600` files) on svc-infra, and are pasted directly into the
  ADO service connection UI — never stored in the repo or in plain chat.
- **Manual approval gates on stg and prod**, none on dev — see Step 4.
  Consider requiring a *different* approver (or more than one) on prod
  than on stg so a single person can't push straight through both.
- **Immutable image promotion, not per-env rebuilds.** The image is built
  and pushed once per pipeline run, tagged with `$(Build.BuildId)`, and
  that exact tag is promoted through dev → stg → prod. `latest` is also
  pushed for convenience browsing in the registry but is never what gets
  deployed — `$(imageTag)` (the immutable build id) is always what the
  Deploy stages reference. This guarantees stg/prod run the exact artifact
  that passed dev, not a rebuild that could drift.
- **Service connections default to per-pipeline authorization** — leave
  "Grant access permission to all pipelines" unchecked in Step 5 so each
  pipeline must be explicitly approved to use the connection the first
  time, rather than every pipeline in the project getting silent access.
- **Production changes require a human running the command.** The RBAC
  bind and token-secret creation for `pipeline-prod` are intentionally
  called out as manual, deliberate steps (Step 2) rather than something
  scripted across environments in one shot — reduces the chance of a
  copy-paste mistake landing in prod.
- **Token rotation.** These are long-lived (non-expiring)
  `kubernetes.io/service-account-token` secrets, needed because CI runs
  non-interactively. If a token is ever suspected compromised, rotate it
  by deleting and recreating the secret (this immediately invalidates the
  old token) and updating the corresponding ADO service connection:
  ```bash
  oc delete secret ado-deployer-token -n <namespace>
  # then re-run Step 2's Secret manifest and Step 3 to rebuild the kubeconfig
  ```

## Status

- [x] `dev` (`ado-pipeline`) — service account, RBAC, token, kubeconfig,
      service connection, and Deploy stage all complete and verified
      (1 pod running).
- [ ] `stg` (`pipeline-stg`) — namespace and service account created;
      RBAC/token/kubeconfig/service connection/environment pending.
- [ ] `prod` (`pipeline-prod`) — namespace and service account created;
      RBAC/token/kubeconfig/service connection/environment pending
      (RBAC step intentionally requires manual execution, not automation,
      per Step 2's production note).
