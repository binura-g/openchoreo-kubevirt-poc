# PoC observations: VM-based components with KubeVirt

Supporting notes for [PROPOSAL.md](PROPOSAL.md): the test environment, how the PoC is implemented, observability
details, the full results and other things noticed along the way. To reproduce it, see [SETUP.md](SETUP.md).


## Test environment and local findings

- **Versions:** the full run (all checks below) was on OpenChoreo `latest-dev` (2026-11-11). It was re-run from
  scratch on **v1.3.3** (Helm charts 1.3.3): platform setup, build, deploy, the dependency VM, staging promotion and the
  zero-downtime rollout. Also KubeVirt v1.9.1, k3d (k3s v1.36.1, single node, arm64), Colima 0.8.4 (vz) on an Apple M3.
- **KubeVirt needs `/dev/kvm`; emulation doesn't work for arm64 guests on arm64.** KubeVirt forces
  `host-passthrough` CPU mode on arm64 and libvirt rejects it without KVM (`CPU mode 'host-passthrough' for aarch64
  qemu domain on aarch64 host is not supported by hypervisor`, kubevirt/kubevirt#11917). Colima's nested
  virtualization (vz, M3+, macOS 15+) fixes it. Ubuntu 24.04 then boots to a running app in ~31 s.
- **virt-handler crash-loops on `inotify: too many open files`** with Colima's default limits; raise
  `fs.inotify.max_user_instances`.
- **The bridge network binding fails on Cilium** (`no gateway address found in routes for eth0`); masquerade works.
- **CDI isn't supported on arm64**, so the PoC uses containerDisks and plain PVCs only.
- **Memory:** budget ~1.3–1.5 GiB of node memory per 1 GiB guest (launcher overhead). KubeVirt requests only 100m
  CPU per vCPU by default (`cpuAllocationRatio: 10`).

## Implementation: platform prerequisites (`openchoreo/00-platform/`)

| File | Purpose |
|---|---|
| `kubevirt-cr.yaml` | KubeVirt v1.9.1 CR (operator from the upstream release URL), `useEmulation: false`. |
| `cluster-agent-kubevirt-rbac.yaml` | Extra ClusterRole/binding giving the data-plane agent access to `kubevirt.io` and `pool.kubevirt.io` kinds, without touching the Helm-managed role. |
| `resource-tree-values.yaml` | Control-plane Helm values adding resource-tree rules for KubeVirt kinds. |

**Resource tree.** The release resource tree walks child edges defined in the openchoreo-api config
([docs](https://openchoreo.dev/docs/platform-engineer-guide/resource-tree-rules/)). The built-in rules cover
Deployment, CronJob, Job and ExternalSecret only, so without new rules the tree stops at `VirtualMachinePool`.
KubeVirt sets controller `ownerReferences` on every hop, so the plain owner-reference matcher is enough:

```
VirtualMachinePool (pool.kubevirt.io/v1beta1)          <- rendered root
└─ VirtualMachine (kubevirt.io/v1)                      ownerRef: VirtualMachinePool
   └─ VirtualMachineInstance (kubevirt.io/v1)           ownerRef: VirtualMachine
      └─ Pod virt-launcher-* (v1)                       ownerRef: VirtualMachineInstance
VirtualMachine (kubevirt.io/v1)                          <- second rule, for types that render a bare VM
└─ VirtualMachineInstance → Pod
```

Applying it is a control-plane `helm upgrade --reuse-values` (see SETUP.md). Notes from doing it:
- openchoreo-api validates the rules at startup and exits on a bad one.
- The authz bootstrap Job is a post-upgrade hook, so it runs again (it is idempotent).
- On a `latest-dev` install, rebuild the installed chart from the Helm release secret rather than pulling
  `latest-dev` again, which could bring unrelated template changes.
- Nodes in the tree have the same "exists = Healthy" status as the release ([Health of non-built-in kinds](PROPOSAL.md#1-health-of-non-built-in-kinds-high)).

## Implementation: build workflow (`openchoreo/01-workflow/`)

- `vm-appdisk-build-cwft.yaml`: Argo `ClusterWorkflowTemplate vm-appdisk-build`. It loads the built image tarball,
  copies `APP_DIR` (default `/app`) out of it, runs `mkfs.ext4 -L ocapp -d`, wraps the result as
  `FROM scratch` / `ADD --chown=107:107 app.img /disk/`, and saves it under the same tag so the stock `publish-image`
  step pushes it. The PoC installs `e2fsprogs` at runtime; it should be baked into the runner image.
- `vm-appdisk-builder.yaml`: `ClusterWorkflow` identical to the stock `dockerfile-builder` (verified against 1.3.3)
  plus a `vm.appDir` parameter and the `build-appdisk` step between build and publish. The repo's `workload.yaml`
  is honoured as usual (endpoints, dependencies).

## Implementation: ComponentType `vm-service` (`openchoreo/02-component-type/`)

Resources rendered per environment:
- `virtualmachinepool`: replicas, vCPUs and memory from `environmentConfigs`; VM labels = `metadata.podSelectors`;
  masquerade interface exposing the endpoint ports; disks for the OS, app, cloud-init and (when there are secrets) secrets.
- `cloudinit`: Secret with `userdata` built in CEL:
  - `/etc/openchoreo/app.env`: config env vars + literal dependency env vars (systemd `EnvironmentFile` syntax);
  - config files at `<mountPath>/<key>`;
  - `openchoreo-bootstrap` script + unit (runs every boot): mounts the app disk read-only at `/opt/app` and the
    secrets disk, writes `/etc/openchoreo/secret.env`, installs secret files with mode 0600, formats and mounts data disks;
  - `openchoreo-app.service`: runs `parameters.startCommand` (or the Workload's command + args), `Restart=always`,
    stdout/stderr to journal and serial console.
- `secrets-external`: one `ExternalSecret` (the data plane's secret store) for all secret env vars and files.
- `service`, `httproute-external`, `httproute-internal`: identical to the stock `service` type.
- `openchoreo.dev/vm-config-hash` annotation on the VM template: a hash of config values, secret refs, dependency
  values, command/args and a `guest-contract/N` marker. Any change alters the template, so the pool replaces the VMs.
  The cloud-init template text itself isn't hashed, so bump `guest-contract/N` when changing it, or running VMs keep
  their old unit files.

**Guest contract** (what the OS image must provide): cloud-init (NoCloud), systemd, ext4 and iso9660 support, and
`/dev/disk/by-id` links. Ubuntu, Fedora and Debian cloud containerDisks qualify. Disk serials: `ocapp` (app),
`ocsecrets` (secrets), `ocd-<instance>` (data disks, ≤ 20 chars).

## Implementation: traits and sample app (`openchoreo/03-traits/`, `04-app/`, `05-zero-downtime-test/`)

- `vm-readiness-probe`: adds an HTTP readiness probe to the VM. KubeVirt copies it to the launcher pod (adding 10 s to
  the initial delay), so Service endpoints follow guest readiness. Given [the health gap](PROPOSAL.md#1-health-of-non-built-in-kinds-high), this is the only real traffic gate.
- `vm-data-disk`: creates a PVC and attaches it as a disk, mounted at `/mnt/<instanceName>`. Data survives VM
  replacement. Single replica only: every replica attaches the same PVC, and QEMU's image lock refuses the second
  VM (`Failed to get "write" lock`), which avoids corruption but leaves that VM failed.
- The stock traits (`persistent-volume-claim`, `health-probes`, HPA/VPA) target Deployment paths and don't apply to
  VMs; VM traits are authored separately, but the trait model itself works unchanged.
- Sample app (`04-app/`): project `vm-poc`; `greeter-vm` built from
  [`openchoreo/sample-workloads/service-go-greeter`](https://github.com/openchoreo/sample-workloads) with both traits,
  a SecretReference to OpenBao, and dev/staging bindings with env, file, secret, sizing and OTLP overrides;
  `greeter-client-vm`, a second VM that depends on greeter-vm's endpoint.
- `05-zero-downtime-test/`: `greeter-vm-ha`, a stateless 2-replica copy used for the rollout test.

## Observability details

OpenChoreo attributes all signals through pod labels (component, project and environment uids). The VM template
carries `metadata.podSelectors`, KubeVirt copies them onto the launcher pod, so every signal from that pod lands on
the right component and environment.

| Signal | How it's collected | VM outcome |
|---|---|---|
| Logs | fluent-bit tails all container logs into OpenSearch | The launcher pod has two log streams: `compute` (libvirt/QEMU noise) and `guest-console-log` (the guest's serial console, on by default in KubeVirt 1.9). With `StandardOutput=journal+console` in the app unit, app lines appear there as `[  33.12] go-greeter[1121]: {json…}`, alongside boot and cloud-init output. Viewing them in the OpenChoreo UI wasn't checked. |
| Metrics | cAdvisor CPU/memory joined with `kube_pod_labels`; requests/limits from kube-state-metrics | Attributed correctly. Values are for the whole VM (≈695 MiB working set for a 1 GiB guest). Memory climbs towards the guest size and doesn't shrink when the app frees memory. |
| Traces | OTel collector with `k8sattributes`, matching on the connection's source IP | Masquerade networking makes guest traffic leave from the launcher pod IP, so spans get the pod's labels. The app still needs `OTEL_EXPORTER_OTLP_ENDPOINT`, as for containers. Readiness probes produce `GET /readyz` spans too. |
| Gateway metrics | Gateway / Cilium, by pod IP | Unchanged. |

Better options for guest logs: an in-guest log shipper with component labels from cloud-init, or a dedicated
virtio-serial/vsock channel for app logs. For metrics, KubeVirt's `kubevirt_vmi_*` series (guest CPU, memory,
network, disk) would need a ServiceMonitor and adapter support.

Not VM-specific: on the `latest-dev` install, OpenCost also exported kube-state-metrics-style series, so every pod
had duplicate `kube_pod_labels` and the adapter's resource numbers were doubled. The v1.3.3 install has no OpenCost.

## Full results

Run on `latest-dev` unless marked otherwise. Guest checks were done over the serial console with the
`debugPassword` set in the binding.

| # | Check | Result |
|---|---|---|
| 1 | KubeVirt v1.9.1 on k3d | ✅ after raising inotify limits; node advertises `devices.kubevirt.io/kvm: 1k` with nested virtualization. Also on v1.3.3. |
| 2 | Guest boot | ❌ under emulation. ✅ under KVM: cloud-init done and app started at ~31 s. |
| 3 | Admission of `vm-service`, traits, workflow | ✅ the webhook type-checks all the CEL. Also on v1.3.3. |
| 4 | Build (`vm-appdisk-builder`) | ✅ 2m40s, all 5 steps; the generated Workload carries the app containerDisk and the repo's endpoints. Also on v1.3.3 (~2.5 min). |
| 5 | autoDeploy → release → data plane | ✅ pool, cloud-init Secret, ExternalSecret, Service, HTTPRoute and PVC rendered and applied. Also on v1.3.3. |
| 6 | Endpoints | ✅ gateway → HTTPRoute → Service → launcher pod → guest: `Hello, Stranger!`. |
| 7 | Config injection | ✅ env values in `/etc/openchoreo/app.env` and in the app process env; config file written; app disk read-only at `/opt/app`. |
| 8 | Secret injection (ESO) | ✅ OpenBao value reaches the app process env; secret file installed with mode 0600. A missing secret blocks the VM from starting, like a pod. |
| 9 | Traits | ✅ readiness gates the endpoints; data disk formatted on first boot, and a file survived VM deletion and recreation. |
| 10 | Dependencies | ✅ the client VM's `app.env` has `GREETER_URL`/`HOST`/`PORT`; `curl $GREETER_URL` from inside it returns `Hello, client-vm!`. Client VM also deployed and running on v1.3.3. |
| 11 | Env sizing / replicas | ✅ replicas, vCPUs and memory follow the binding; replicas 0 scales the pool in. |
| 12 | Ready status | ❌ binding Ready while the VM was still scheduling; staging Ready within 10 s, before any VM booted, and still Ready with one of two VMs failed. |
| 13 | Pool stability | ✅ no reconcile churn across agent re-applies. |
| 14 | Rollout on config change | ✅ binding or ComponentType changes replace the VM with the new config. 1 replica: ~60 s of 503s. Rebuild-triggered rollout not tested (it changes the template the same way). |
| 15 | Promotion to staging | ✅ same ComponentRelease, staging config, own PVC and ExternalSecret: `Hello, staging!`. Promotion also applied on v1.3.3. |
| 16 | Replicas > 1 with a data disk | ❌ second VM fails on QEMU's write lock; the pool removed the failed VM on scale-in. |
| 17 | Logs | ✅ app stdout/stderr in `guest-console-log`, with a kernel-timestamp prefix. |
| 18 | Metrics | ✅ metrics adapter returns CPU/memory usage, requests and limits for the component. |
| 19 | Traces | ✅ tracing adapter returns the component's spans by component/environment uid. |
| 20 | Zero-downtime rollout (v1.3.3) | ✅ `greeter-vm-ha`, 2 replicas, readiness probe every 2 s, no data disk. A config change replaced both VMs one at a time while curl hit the gateway ~7 req/s: **2,005 requests over 4.5 min, all 200**. VM-1 stopped 05:00:42 → new one Ready 05:02:54 → only then VM-0 stopped (05:02:58) → Ready 05:04:34. Not tested: concurrent or long-lived connections, multiple nodes, in-flight requests during guest shutdown under load. |

## Other observations

- A Project applied with kubectl gets a ProjectRelease but no ProjectReleaseBindings, so the data-plane namespace
  doesn't exist until one is created per environment. Not VM-specific.
- ProjectReleases have no project label; read the generated name from the Project's `status.latestRelease.name`.
- The secrets disk uses the env var name (`API_KEY`) as the file name, not the SecretReference key (`api-key`),
  because the CEL context exposes the env name plus the remote ref.
- Comments inside CEL `${…}` blocks must use `//`, not `#`.
- Applying a binding manifest over an autoDeploy-created ReleaseBinding warns about a missing
  `last-applied-configuration` annotation; it's harmless.
