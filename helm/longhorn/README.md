# Longhorn (technical notes)

Persistent block storage for the cluster, deployed by Argo CD from this repo.

| | |
|---|---|
| Chart | `longhorn` from `https://charts.longhorn.io` |
| Version | `1.13.0` (pinned in `manifests/argocd/longhorn.yaml`) |
| Namespace | `longhorn-system` |
| Overrides | `helm/longhorn/values.yaml` (this folder) |
| Argo CD app | `manifests/argocd/longhorn.yaml`, sync-wave `1` |
| UI | `http://192.168.49.202` (MetalLB `LoadBalancer`, no authentication) |
| Environment | single-node minikube (Docker driver), node `minikube` `192.168.49.2`, Debian 12 |

## 1. What Longhorn does

Longhorn turns local disk on cluster nodes into Kubernetes `PersistentVolume`s. For every volume it runs a dedicated **engine** process that exposes the volume as an iSCSI block device, plus one or more **replicas**, which are sparse files on the node's disk. A pod's PVC is attached to the node as an iSCSI device, formatted (ext4) and mounted into the pod.

This is why the node needs `iscsiadm`: the kubelet and Longhorn attach every volume over iSCSI, even when the volume is on the same machine.

## 2. Components (pods in `longhorn-system`)

| Pod | What it does |
|-----|--------------|
| `longhorn-manager` (DaemonSet, 1 per node) | The control plane on each node. Reconciles Longhorn CRDs (volumes, nodes, settings), manages replicas and talks to the engines. On start it runs an environment check, which is what failed without `open-iscsi`. |
| `longhorn-global-manager` (3 replicas) | Cluster-wide singleton-style work that doesn't belong to a single node. |
| `instance-manager` | Hosts the engine and replica processes for volumes on its node. |
| `engine-image-*` | Delivers the engine binary to each node. |
| `longhorn-driver-deployer` | Deploys the CSI components once the managers are healthy. It waits in `Init` until then. |
| `csi-attacher`, `csi-provisioner`, `csi-resizer`, `csi-snapshotter` | The Kubernetes CSI sidecars that attach, create, expand and snapshot volumes. |
| `longhorn-csi-plugin` | The per-node CSI driver that mounts volumes. |
| `longhorn-ui` (2 replicas) | The web UI, exposed by the `longhorn-frontend` service. |

StorageClasses created: `longhorn` (dynamic provisioning) and `longhorn-static` (for pre-existing volumes).

## 3. Repo wiring

Longhorn follows the same two-source pattern as the other Helm apps:

```
bootstrap/bootstrap.yaml            root app "argo-objects"
  -> manifests/argocd/longhorn.yaml child Application
       source 1: charts.longhorn.io  chart longhorn @ 1.13.0
       source 2: this repo (ref: values) -> $values/helm/longhorn/values.yaml
```

Relevant Application settings:
- `sync-wave: "1"`: installs after MetalLB (wave 0), so the UI service can get an IP.
- `CreateNamespace=true`: Argo creates `longhorn-system`.
- `ServerSideApply=true`: Longhorn's CRDs are large and would exceed the client-side apply annotation limit.
- `automated.prune` and `selfHeal`: anything removed from git is removed from the cluster, and manual drift is reverted.

## 4. Values explained (`values.yaml`)

```yaml
service:
  ui:
    type: LoadBalancer
```
Makes `longhorn-frontend` a `LoadBalancer`, so MetalLB assigns it an address from `homelab-pool` (`192.168.49.200-250`). This matches the goal of exposing everything through MetalLB. The Longhorn UI has no login, so anyone who can reach the IP can manage volumes. Keep it on the LAN.

```yaml
persistence:
  defaultClassReplicaCount: 1
defaultSettings:
  defaultReplicaCount: 1
```
The chart default is 3 replicas per volume. With one node there is nowhere to place 3, so volumes would stay `degraded`. These two keys cover different things:
- `persistence.defaultClassReplicaCount` sets `numberOfReplicas` on the `longhorn` StorageClass (verified: `numberOfReplicas: "1"`).
- `defaultSettings.defaultReplicaCount` sets Longhorn's own default for volumes created through the UI or API (verified: `default-replica-count` is `1`).

With 1 replica there is **no redundancy**. If the node's disk is lost, so is the data. When you add nodes, raise both to 2 or 3.

## 5. Node prerequisites

Longhorn needs these on every node, as the host (the minikube container), not inside a pod:

- `open-iscsi` (provides `iscsiadm`, and the `iscsid` daemon)
- `nfs-common` (only for RWX volumes, but Longhorn's check warns without it)
- the `iscsi_tcp` kernel module (shares the host kernel, already loaded)

Installed with:

```sh
minikube ssh -- "sudo apt-get update && sudo apt-get install -y open-iscsi nfs-common && sudo systemctl enable --now iscsid"
```

Caveat: inside the minikube container `apt` printed `policy-rc.d returned 101, not running 'start iscsid'`, so the install did not start the daemon. It had to be started manually with `sudo systemctl start iscsid`.

**Important:** this lives in the node container's filesystem. `minikube delete` (or recreating the node) removes it, along with all Longhorn data. Redo this step after any rebuild.

## 6. How it fails when the prerequisite is missing

`longhorn-manager` runs this on startup, using `nsenter` to enter the host's namespaces:

```
nsenter --mount=/host/proc/<pid>/ns/mnt --net=/host/proc/<pid>/ns/net iscsiadm --version
```

If `iscsiadm` isn't on the host, it exits `127` and the manager dies with `failed to check environment, please make sure you have iscsiadm/open-iscsi installed on the host`. The pod goes `CrashLoopBackOff` (`1/2`), `longhorn-driver-deployer` stays in `Init:0/1`, and no CSI pods or settings objects are ever created.

## 7. Incident log: the stuck deletion

During setup, Longhorn was removed from git and then restored:

1. Commit `2f00b78` replaced Longhorn with minikube's `standard` storage and deleted the Application.
2. Argo CD synced that commit and started deleting the `longhorn` app.
3. Longhorn chart ships a **pre-delete hook**, the `longhorn-uninstall` job. It refuses to run unless the Longhorn setting `deleting-confirmation-flag` is `true`, so it failed, and the app got stuck with Argo's `pre-delete-finalizer` still attached (condition `DeletionError`).
4. The restore (revert commit `264d4ce`) was not picked up, because `argo-objects` was still on the old revision (`OutOfSync`).
5. The `deleting-confirmation-flag` setting did not even exist, since the crashed manager never created it.

Resolution:
1. Installed and started `open-iscsi` on the node.
2. Deleted the crashed manager pod so the replacement started cleanly, which created the settings.
3. Removed the finalizers from the `longhorn` Application so Argo dropped it without running the uninstall. The workloads were healthy and held no data.
4. Hard-refreshed `argo-objects` (`argocd.argoproj.io/refresh=hard`), which recreated `longhorn` from the latest commit and removed `kube-prometheus-stack`.

Lesson: to remove Longhorn properly, set the flag first with `kubectl -n longhorn-system patch settings.longhorn.io deleting-confirmation-flag --type=merge -p '{"value":"true"}'`, then delete the app.

## 8. Known issues and follow-ups

- **Two default StorageClasses:** `longhorn` and minikube's `standard` are both `(default)`. A PVC that doesn't name a class may land on either. Fix with `kubectl patch sc standard -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'` or `minikube addons disable default-storageclass`.
- **App shows `OutOfSync` while `Healthy`:** likely diff noise on generated Longhorn resources. Not investigated yet.
- **No redundancy** with a single replica (see section 4).
- **Unauthenticated UI** (see section 4).

## 9. Verify it works

```sh
kubectl -n longhorn-system get pods        # all Running
kubectl get sc                             # longhorn present
kubectl -n longhorn-system get svc longhorn-frontend   # EXTERNAL-IP 192.168.49.202
```

Quick functional test with a throwaway PVC:

```sh
kubectl apply -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: longhorn-test}
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn
  resources: {requests: {storage: 1Gi}}
EOF
kubectl get pvc longhorn-test       # should reach Bound
kubectl delete pvc longhorn-test
```

## 10. Next step

kube-prometheus-stack was removed for now. Its saved values (`helm/kube-prometheus-stack/values.yaml`) already point Grafana, Prometheus and Alertmanager at the `longhorn` StorageClass and expose Grafana through MetalLB. Re-add `manifests/argocd/kube-prometheus-stack.yaml` with `sync-wave: "2"` once Longhorn is confirmed working.
