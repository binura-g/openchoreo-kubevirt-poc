# Setup: KubeVirt VM components on OpenChoreo (local k3d)

Tested end to end with **OpenChoreo v1.3.3** (Helm charts `1.3.3`, installed with the k3d quick-start script), not
`latest-dev`. Also: KubeVirt v1.9.1, Colima 0.8.4 (vz), k3d 5.8.3 (k3s v1.36.1), Apple M3 Max.

## 0. Prerequisites

- **Apple Silicon M3 or newer, macOS 15 or newer.** Apple's Virtualization.framework only supports nested
  virtualization on these. On an M1 or M2 there is no `/dev/kvm`, and KubeVirt cannot boot arm64 guests at all:
  software emulation fails with `CPU mode 'host-passthrough' ... not supported by hypervisor` (see the test environment notes in POC_OBSERVATIONS.md).
- CLI tools: `colima`, `docker`, `k3d`, `kubectl`, `helm`, `jq`, `yq`, `expect` (`expect` is only needed for guest checks).
- OpenChoreo v1.3+ installed on a k3d cluster named `openchoreo` in the Colima Docker runtime. Use installation script from https://openchoreo.dev/docs/getting-started/try-it-out/on-k3d-locally/
  Starting from scratch? Do §1 (Colima) **before** installing OpenChoreo, so the cluster is created on a Colima VM
  that already has `/dev/kvm` and you avoid a restart later.
- About 3 GiB of free memory on the node for the VMs in this guide. Each 1 GiB guest uses about 1.3–1.5 GiB including the launcher overhead.

---

## 1. Colima with nested virtualization

KubeVirt needs `/dev/kvm` inside the k3d node container, so Colima must run with nested virtualization.

```bash
# Existing profile (must already be vm-type vz; vm-type cannot be changed on an existing instance)
# You can use a cpu/memory amount that works for your machine, it doesn't need to be this high.
colima stop
colima start --vm-type vz --nested-virtualization --cpu 8 --memory 15
```

To persist these settings, set `nestedVirtualization: true` in `~/.colima/default/colima.yaml` (along with `vmType: vz`).

Check `/dev/kvm` in the Colima VM and in the k3d node:
```bash
colima ssh -- ls -l /dev/kvm
docker exec k3d-openchoreo-server-0 ls -l /dev/kvm
k3d cluster start openchoreo        # only if the cluster did not come back by itself
kubectl get nodes
```

### 1.1 Raise inotify limits

The Colima default `fs.inotify.max_user_instances=128` is used up by the OpenChoreo stack, and `virt-handler` then
crash-loops with `too many open files`. Increase it:

```bash
colima ssh -- sudo sysctl -w fs.inotify.max_user_instances=8192 fs.inotify.max_user_watches=1048576
```

To keep it across restarts, add this to `~/.colima/default/colima.yaml`:
```yaml
provision:
  - mode: system
    script: sysctl -w fs.inotify.max_user_instances=8192 fs.inotify.max_user_watches=1048576
```

---

## 2. KubeVirt

```bash
# Operator
kubectl apply -f https://github.com/kubevirt/kubevirt/releases/download/v1.9.1/kubevirt-operator.yaml
kubectl -n kubevirt rollout status deploy/virt-operator --timeout=300s

# KubeVirt CR (useEmulation: false, so it requires /dev/kvm)
kubectl apply -f openchoreo/00-platform/kubevirt-cr.yaml
kubectl -n kubevirt wait kv kubevirt --for condition=Available --timeout=600s
```

Check that the node advertises KVM. It should show `1k`. If you see `0` it means no `/dev/kvm`, so go back to (1) and check Colima settings.
```bash
kubectl get node -o jsonpath='{.items[0].status.allocatable.devices\.kubevirt\.io/kvm}{"\n"}'
```

If `virt-handler` is in CrashLoopBackOff, apply step 1.1 and then run
`kubectl -n kubevirt delete pod -l kubevirt.io=virt-handler`.

Notes:
- If applying the CR right after a restart fails with `no endpoints available for service "kubevirt-operator-webhook"`,
  wait for `virt-operator` and retry.
- Use the **masquerade** network binding. The provided component type `vm-service` already does (the bridge binding fails on Cilium).
- CDI is not installed as it isn't supported on arm64. This PoC uses containerDisks plus plain PVCs.

---

## 3. OpenChoreo platform prerequisites

### 3.1 Cluster-agent RBAC for KubeVirt kinds

The data-plane cluster agent's ClusterRole is an allow-list. This adds `kubevirt.io` and `pool.kubevirt.io` without
editing the Helm-managed role.

```bash
kubectl apply -f openchoreo/00-platform/cluster-agent-kubevirt-rbac.yaml
kubectl auth can-i create virtualmachinepools.pool.kubevirt.io --as=system:serviceaccount:openchoreo-data-plane:cluster-agent-dataplane -A # → yes
```

### 3.2 Release resource-tree rules (optional: Allows the UI to show the full component k8s resource tree for VM-based components)

[`openchoreo/00-platform/resource-tree-values.yaml`](openchoreo/00-platform/resource-tree-values.yaml) is a
**control-plane Helm values file**, not a manifest. Upgrade the installed release with `--reuse-values`, pinned to
the **installed chart version** so nothing else changes:

```bash
CP_VERSION=$(helm -n openchoreo-control-plane list -f '^openchoreo-control-plane$' -o json | jq -r '.[0].chart' | sed 's/^openchoreo-control-plane-//')
echo "$CP_VERSION"     # e.g. 1.3.3

# Dry run: the rendered config should show the new rules (VirtualMachinePool, VirtualMachine)
helm upgrade openchoreo-control-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-control-plane \
  --version "$CP_VERSION" -n openchoreo-control-plane --reuse-values \
  -f openchoreo/00-platform/resource-tree-values.yaml --dry-run=server | grep -n 'kind: VirtualMachine'

helm upgrade openchoreo-control-plane oci://ghcr.io/openchoreo/helm-charts/openchoreo-control-plane \
  --version "$CP_VERSION" -n openchoreo-control-plane --reuse-values \
  -f openchoreo/00-platform/resource-tree-values.yaml --wait --timeout 10m
kubectl -n openchoreo-control-plane rollout status deploy/openchoreo-api
kubectl -n openchoreo-control-plane get cm openchoreo-api-config -o jsonpath='{.data.config\.yaml}' \
  | yq '.resource_tree.rules[].root.kind'      # should include VirtualMachinePool and VirtualMachine

# Roll back if needed: helm rollback openchoreo-control-plane <previous revision> -n openchoreo-control-plane
```

<details>
<summary>Control plane installed from the mutable <code>latest-dev</code> chart instead</summary>

Pulling `latest-dev` again could bring unrelated template changes, so rebuild the exact installed chart from the
Helm release secret and upgrade from that:

```bash
REV=$(helm -n openchoreo-control-plane history openchoreo-control-plane -o json | jq '.[-1].revision')
kubectl -n openchoreo-control-plane get secret sh.helm.release.v1.openchoreo-control-plane.v$REV \
  -o jsonpath='{.data.release}' | base64 -d | base64 -d | gunzip > release.json
mkdir -p cp-chart
jq '.chart.metadata' release.json | yq -P > cp-chart/Chart.yaml
jq '.chart.values'   release.json | yq -P > cp-chart/values.yaml
jq -r '.chart.schema' release.json | base64 -d > cp-chart/values.schema.json
python3 -I -c 'import json,base64,os
c=json.load(open("release.json"))["chart"]
for k in ("templates","files"):
    for f in c.get(k) or []:
        p=os.path.join("cp-chart",f["name"]); os.makedirs(os.path.dirname(p),exist_ok=True)
        open(p,"wb").write(base64.b64decode(f["data"]))'

# Dry run: the diff should only show the new rules and the config checksum
helm upgrade openchoreo-control-plane ./cp-chart -n openchoreo-control-plane --reuse-values \
  -f openchoreo/00-platform/resource-tree-values.yaml --dry-run=server

helm upgrade openchoreo-control-plane ./cp-chart -n openchoreo-control-plane --reuse-values \
  -f openchoreo/00-platform/resource-tree-values.yaml --wait
kubectl -n openchoreo-control-plane rollout status deploy/openchoreo-api
kubectl -n openchoreo-control-plane get cm openchoreo-api-config -o jsonpath='{.data.config\.yaml}' \
  | yq '.resource_tree.rules[].root.kind'
```
</details>

openchoreo-api checks the rules at startup and exits if one is invalid, so keep an eye on the helm rollout. For more detail, see
the resource-tree notes in POC_OBSERVATIONS.md.

### 3.3 Build workflow, traits and ComponentType

```bash
kubectl apply -f openchoreo/01-workflow/vm-appdisk-build-cwft.yaml   # Argo ClusterWorkflowTemplate (build-appdisk step)
kubectl apply -f openchoreo/01-workflow/vm-appdisk-builder.yaml      # OpenChoreo ClusterWorkflow
kubectl apply -f openchoreo/03-traits/
kubectl apply -f openchoreo/02-component-type/vm-service.yaml

kubectl get clusterworkflowtemplate vm-appdisk-build
kubectl get clusterworkflow vm-appdisk-builder
kubectl get clustertrait vm-readiness-probe vm-data-disk
kubectl get clustercomponenttype vm-service
```

Glossary:

| Manifest | Purpose |
|---|---|
| `vm-appdisk-build-cwft.yaml` | Argo step that turns the built image's `APP_DIR` into an ext4 disk wrapped in a scratch containerDisk image. |
| `vm-appdisk-builder.yaml` | Steps: checkout → Dockerfile build → **build-appdisk** → publish → generate Workload. |
| `vm-readiness-probe.yaml` | Adds a VMI HTTP readiness probe, which gates the Service endpoints. |
| `vm-data-disk.yaml` | PVC plus a disk mounted at `/mnt/<instance>`. **Single replica only.** |
| `vm-service.yaml` | `proxy`-type ComponentType: VirtualMachinePool, cloud-init Secret, ExternalSecret, Service and HTTPRoutes. |

If you change the cloud-init template in `vm-service.yaml`, bump the `guest-contract/N` literal in the
`openchoreo.dev/vm-config-hash` expression. Otherwise running VMs keep their old unit files.

---

## 4. Project and data-plane namespaces

```bash
kubectl apply -f openchoreo/04-app/01-project.yaml

# A Project applied with kubectl gets a ProjectRelease but no ProjectReleaseBinding, so the dataplane
# namespace is never created. Bind it explicitly using the generated ProjectRelease name:
PR=$(kubectl -n default get project vm-poc -o jsonpath='{.status.latestRelease.name}')
sed "s/projectRelease: .*/projectRelease: $PR/" openchoreo/04-app/01b-projectreleasebinding-development.yaml | kubectl apply -f -
kubectl get ns | grep dp-default-vm-poc-development
```

---

## 5. Deploy the sample app (greeter-vm)

The app is `service-go-greeter` from https://github.com/openchoreo/sample-workloads. It is built with its Dockerfile,
and its `/app` directory is packaged as the VM's app disk. Inside an Ubuntu 24.04 guest, systemd runs `/opt/app/go-greeter`.

### 5.1 Component

```bash
kubectl apply -f openchoreo/04-app/03-component-greeter-vm.yaml
```

### 5.2 Add a secret for the component to consume (as an example)

```bash
kubectl -n openbao exec openbao-0 -- env BAO_TOKEN=root \
  bao kv put secret/vm-poc-greeter api-key=s3cr3t-from-openbao tls.crt=dummy-cert
kubectl apply -f openchoreo/04-app/02-secret-reference.yaml
```

### 5.3 Build the VM based component

```bash
kubectl create -f openchoreo/04-app/04-workflowrun-greeter-vm.yaml      # generateName, so `create`, not `apply`
kubectl -n default get workflowrun -l openchoreo.dev/component=greeter-vm -w
# Done when WorkflowSucceeded=True (~3 min):
kubectl -n default get workflowrun -l openchoreo.dev/component=greeter-vm \
  -o jsonpath='{range .items[*]}{.metadata.name} {.status.conditions[?(@.type=="WorkflowSucceeded")].status}{"\n"}{end}'
kubectl -n default get workload greeter-vm-workload -o jsonpath='{.spec.container.image}{"\n"}'
```

Argo pods run in `workflows-default`. Because the Component has `autoDeploy: true`, a successful build creates a
ComponentRelease and the `greeter-vm-development` ReleaseBinding.

### 5.4 Development environment configuration

[`05-releasebinding-greeter-vm-development.yaml`](openchoreo/04-app/05-releasebinding-greeter-vm-development.yaml) sets:
- VM size: 1 vCPU, 1 GiB memory, 1 replica
- `debugPassword`, which lets you log in on the serial console (PoC only)
- trait settings: data disk size and probe delay
- env values, a secret env, a config file and a secret file
- the OTLP endpoint for traces

Apply it after the build, so the binding already has its `releaseName`. Expected side effects:
- kubectl warns that the binding `is missing the kubectl.kubernetes.io/last-applied-configuration annotation`
  (autoDeploy created it). Harmless: the fields not in the file, such as `releaseName`, are kept.
- autoDeploy has already started a VM with the default config. The new config changes the VM's config hash, so the
  pool replaces that VM: you will briefly see the VMI `Failed` and the VM `CrashLoopBackOff` before the new one starts.
- The new VM stays in `Scheduling` (launcher pod `Init:0/3`) until the OpenBao value from §5.2 exists.

```bash
kubectl apply -f openchoreo/04-app/05-releasebinding-greeter-vm-development.yaml
NS=$(kubectl get ns -o name | grep dp-default-vm-poc-development | cut -d/ -f2)
kubectl -n $NS get vmpool,vm,vmi,pod,pvc,externalsecret,svc,httproute
kubectl -n $NS wait vmi --all --for condition=Ready --timeout=300s
```

The boot sequence is: the pod pulls the containerDisks, the guest boots, cloud-init runs, `openchoreo-bootstrap`
mounts the disks and writes the secrets, then `openchoreo-app` starts. This takes about 30 s under KVM. The Ready
condition follows the `/readyz` probe.

### 5.5 Dependency consumer (greeter-client-vm)

This is a second VM component with a hand-written Workload. It reuses greeter-vm's app disk and depends on
`greeter-vm/greeter-api`. Set the Workload image to the one from §5.3, then:

```bash
IMG=$(kubectl -n default get workload greeter-vm-workload -o jsonpath='{.spec.container.image}')
sed "s|image: .*|image: $IMG|" openchoreo/04-app/06-component-greeter-client-vm.yaml | kubectl apply -f -
sleep 10   # wait for autoDeploy to create the binding
kubectl -n default patch releasebinding greeter-client-vm-development --type merge \
  -p '{"spec":{"componentTypeEnvironmentConfigs":{"replicas":1,"memory":"512Mi","debugPassword":"openchoreo"}}}'
```

([`07-releasebinding-greeter-client-vm-development.yaml`](openchoreo/04-app/07-releasebinding-greeter-client-vm-development.yaml)
records the same patch. Set `replicas: 0` to free the memory.) As in §5.4, the patch replaces the VM that autoDeploy
started, so expect one `Failed` VMI first. The client has no secrets, so it does not wait on OpenBao.

### 5.6 Promote to staging

```bash
PR=$(kubectl -n default get project vm-poc -o jsonpath='{.status.latestRelease.name}')
sed "s/projectRelease: .*/projectRelease: $PR/" openchoreo/04-app/08-projectreleasebinding-staging.yaml | kubectl apply -f -

REL=$(kubectl -n default get releasebinding greeter-vm-development -o jsonpath='{.spec.releaseName}')
sed "s/releaseName: .*/releaseName: $REL/" openchoreo/04-app/09-releasebinding-greeter-vm-staging.yaml | kubectl apply -f -
SNS=$(kubectl get ns -o name | grep dp-default-vm-poc-staging | cut -d/ -f2)
kubectl -n $SNS wait vmi --all --for condition=Ready --timeout=300s
```

Keep `replicas: 1` in staging. The `vm-data-disk` PVC is shared by every replica, so a second VM fails on QEMU's
disk write lock (PROPOSAL.md, note [3] under the summary table).

---

## 6. Verify

### 6.1 Traffic, readiness, dependency
```bash
curl "http://development-default.openchoreoapis.localhost:19080/greeter-vm-greeter-api/greeter/greet"              # Hello, Stranger!
curl "http://staging-default.openchoreoapis.localhost:19080/greeter-vm-greeter-api/greeter/greet?name=staging"     # Hello, staging!
kubectl -n $NS get endpointslices                                   # launcher pod IP only once /readyz answers
kubectl -n default get releasebinding greeter-vm-development \
  -o jsonpath='{range .status.conditions[*]}{.type}={.status} {.reason}{"\n"}{end}'
```
The ReleaseBinding reports Ready as soon as the pool exists, whether or not the guest is up (PROPOSAL.md, "Health of non-built-in kinds").
To see the real state, use the VMI Ready condition or the endpoints.

### 6.2 Inside the guest (serial console)
`virtctl console` works if you have `virtctl`. Without it, use [`scripts/guest-console.exp`](scripts/guest-console.exp).
It runs `nc` against the serial socket in the `compute` container and logs in as `ubuntu` with `debugPassword`:

```bash
POD=$(kubectl -n $NS get pod -l kubevirt.io=virt-launcher,openchoreo.dev/component=greeter-vm -o jsonpath='{.items[0].metadata.name}')
cat > /tmp/cmds.txt <<'EOF'
sudo cat /etc/openchoreo/app.env
sudo cat /etc/openchoreo/secret.env
sudo cat /etc/greeter/app.properties /etc/greeter/tls/tls.crt
findmnt -n /opt/app
findmnt -n /mnt/data
systemctl is-active openchoreo-app
systemd-detect-virt
EOF
expect scripts/guest-console.exp $NS $POD /tmp/cmds.txt
```

Dependency call from the client VM:
```bash
CPOD=$(kubectl -n $NS get pod -l kubevirt.io=virt-launcher,openchoreo.dev/component=greeter-client-vm -o jsonpath='{.items[0].metadata.name}')
printf '. <(sudo cat /etc/openchoreo/app.env); curl -s "$GREETER_URL/greeter/greet?name=client-vm"\n' > /tmp/c.txt
expect scripts/guest-console.exp $NS $CPOD /tmp/c.txt          # Hello, client-vm!
```

### 6.3 Rollout on config change
Change any env value in `05-…development.yaml` and re-apply. The pool stops the VM and recreates it with the new
cloud-init. With 1 replica, expect about 60 s of 503s.
```bash
kubectl -n $NS get vmi -w
```

### 6.4 Observability
```bash
# Logs: the app's stdout/stderr go to the serial console, streamed by the guest-console-log container
kubectl -n $NS logs $POD -c guest-console-log | grep go-greeter

# Traces: the tracing adapter, queried by uid
C=$(kubectl -n default get component greeter-vm -o jsonpath='{.metadata.uid}')
E=$(kubectl -n default get environment development -o jsonpath='{.metadata.uid}')
P=$(kubectl -n default get project vm-poc -o jsonpath='{.metadata.uid}')
kubectl -n openchoreo-observability-plane port-forward svc/tracing-adapter 19100:9100 &
curl -s -X POST localhost:19100/api/v1alpha1/traces/query -H 'Content-Type: application/json' -d "{
  \"startTime\":\"$(date -u -v-15M +%FT%TZ)\",\"endTime\":\"$(date -u +%FT%TZ)\",\"limit\":5,
  \"searchScope\":{\"namespace\":\"default\",\"project\":\"$P\",\"component\":\"$C\",\"environment\":\"$E\"}}"

# Metrics: the metrics adapter (same scope)
kubectl -n openchoreo-observability-plane port-forward svc/metrics-adapter 19099:9099 &
curl -s -X POST localhost:19099/api/v1/metrics/query -H 'Content-Type: application/json' -d "{
  \"metric\":\"resource\",\"startTime\":\"$(date -u -v-10M +%FT%TZ)\",\"endTime\":\"$(date -u +%FT%TZ)\",\"step\":\"1m\",
  \"searchScope\":{\"namespace\":\"default\",\"project\":\"$P\",\"component\":\"$C\",\"environment\":\"$E\"}}"
```
In the UI, logs, metrics and traces appear under the greeter-vm component in each environment. If your install also
runs OpenCost (the `latest-dev` one did; v1.3.3 doesn't), it exports duplicate kube-state-metrics series and the
resource numbers come out doubled (POC_OBSERVATIONS.md, observability details).


### 6.5 Zero-downtime rollout (separate test component)

`greeter-vm` can't show this: its data disk caps it at 1 replica, so every rollout has a ~60 s gap. The test in
[`openchoreo/05-zero-downtime-test/`](openchoreo/05-zero-downtime-test/) deploys `greeter-vm-ha`, a stateless copy:
- it reuses greeter-vm's app disk (no build) and has no secrets
- it has a readiness probe and no data disk
- it runs 2 replicas

```bash
IMG=$(kubectl -n default get workload greeter-vm-workload -o jsonpath='{.spec.container.image}')
sed "s|image: .*|image: $IMG|" openchoreo/05-zero-downtime-test/01-component-greeter-vm-ha.yaml | kubectl apply -f -
sleep 10
kubectl apply -f openchoreo/05-zero-downtime-test/02-releasebinding-greeter-vm-ha-development.yaml
kubectl -n $NS wait vmi -l openchoreo.dev/component=greeter-vm-ha --for condition=Ready --timeout=600s
```

Send steady traffic, trigger a rollout by changing `ROLLOUT_MARKER`, and stop once both VMIs are new and Ready
(about 4–5 min):

```bash
URL="http://development-default.openchoreoapis.localhost:19080/greeter-vm-ha-greeter-api/greeter/greet"
( while :; do echo "$(date +%T) $(curl -s -o /dev/null -m 2 -w '%{http_code}' "$URL")"; sleep 0.1; done ) > /tmp/zd.log &
LOOP=$!
kubectl -n default patch releasebinding greeter-vm-ha-development --type json \
  -p '[{"op":"replace","path":"/spec/workloadOverrides/container/env/0/value","value":"'$(date +%s)'"}]'
kubectl -n $NS get vmi -l openchoreo.dev/component=greeter-vm-ha -w     # Ctrl-C when both are new and Ready
kill $LOOP
awk '{c[$2]++} END {for (k in c) print k, c[k]}' /tmp/zd.log           # expect only 200
```

Expected result: only `200` responses. The pool replaces one VM, waits until the new one passes its readiness
probe, then replaces the other (POC_OBSERVATIONS.md, full results #20). With `replicas: 1`, the same test shows about 60 s of `503`.
Remove the test with `kubectl -n default delete component greeter-vm-ha` and
`kubectl -n default delete workload greeter-vm-ha-workload`.

---

## 7. Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Node `devices.kubevirt.io/kvm: 0` | No `/dev/kvm`: Colima isn't running with `--nested-virtualization` (§1). |
| `virt-handler` CrashLoopBackOff, `too many open files` | inotify limits (§1.1). |
| VMI `Scheduling`, launcher pod `Init:0/3`, `FailedMount ... secret ... not found` | ExternalSecret not synced: OpenBao value missing (§5.2). Once it exists, ESO picks it up on the next refresh, or force it: `kubectl -n $NS annotate externalsecret --all force-sync=$(date +%s) --overwrite`. |
| ProjectRelease lookup returns nothing (`array index out of bounds`) | ProjectReleases have no project label. Read the name from `kubectl -n default get project vm-poc -o jsonpath='{.status.latestRelease.name}'` (§4). |
| Everything breaks after `colima stop/start` | Re-apply the inotify limits (§1.1) unless persisted, and re-seed OpenBao (§5.2): it runs in dev mode and loses app secrets on restart. |
| `no healthy upstream` / 503 from the gateway | No ready endpoints. The VM is still booting, failed the readiness probe, or is being replaced by a rollout. |
| Second replica VMI `Failed`, `Failed to get "write" lock` | The `vm-data-disk` PVC is shared across replicas. Use `replicas: 1`. |
| `namespaces "dp-default-vm-poc-…" not found` on a binding | Missing ProjectReleaseBinding for that environment (§4, §5.6). |
| ComponentType apply: `Syntax error: token recognition error at: '#'` | Comments inside CEL `${…}` must use `//`, not `#`. |
| Template change did not reach running VMs | Bump `guest-contract/N` in `vm-service.yaml` (§3.3). |

---

## 8. Teardown

```bash
kubectl delete -f openchoreo/04-app/09-releasebinding-greeter-vm-staging.yaml -f openchoreo/04-app/08-projectreleasebinding-staging.yaml
kubectl -n default delete component greeter-client-vm greeter-vm
kubectl -n default delete workload greeter-client-vm-workload --ignore-not-found
kubectl delete -f openchoreo/04-app/02-secret-reference.yaml -f openchoreo/04-app/01b-projectreleasebinding-development.yaml \
  -f openchoreo/04-app/01-project.yaml
kubectl delete -f openchoreo/02-component-type/ -f openchoreo/03-traits/ -f openchoreo/01-workflow/
kubectl delete -f openchoreo/00-platform/cluster-agent-kubevirt-rbac.yaml
kubectl delete -f openchoreo/00-platform/kubevirt-cr.yaml        # wait until the KubeVirt CR is gone
kubectl delete -f https://github.com/kubevirt/kubevirt/releases/download/v1.9.1/kubevirt-operator.yaml
# Resource-tree rules: helm rollback openchoreo-control-plane <revision before §3.2> -n openchoreo-control-plane
```
