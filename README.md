# MySQL Stateful DR with ACM and VolSync

> **Disclaimer:** This demo is for learning and testing purposes only.
> All configurations should be thoroughly tested in a non-production
> environment before applying to any production cluster.

## How VolSync replication works

For this repository exercise we will use VolSync with the **rsync-TLS** mover, for 1:1 cross-cluster PVC replication.
Replication is **push-based**: the `ReplicationSource` on cluster1 drives the schedule (cron). The `ReplicationDestination` on cluster2 is passive: it creates a LoadBalancer Service, waits for incoming data, and storesVolumeSnapshots.

### VolSync addon - rsync-TLS replication and failover

```
  REPLICATION (continuous, every 3 min)

  cluster1 (source)                                cluster2 (destination)
 ┌───────────────────────────────────┐            ┌───────────────────────────────────┐
 │                                   │            │                                   │
 │  MySQL ──► PVC (mysql-pv-claim)   │            │  ReplicationDestination           │
 │                │                  │            │    serviceType: LoadBalancer      │
 │                ▼                  │            │    copyMethod: Snapshot           │
 │  ReplicationSource                │            │                                   │
 │    schedule: */3 * * * *          │            │                                   │
 │    copyMethod: Snapshot           │            │                                   │
 │         │                         │            │                                   │
 │         ▼                         │  rsync-TLS │                                   │
 │VolumeSnapshot ──► (temp PVC) ────────(deltas)─────► LB Service (rsync receiver)    │
 │                                   │            │         │ (aux PVC)               │
 │                                   │            │         ▼                         │
 │                                   │            │  VolumeSnapshot (latestImage)     │
 │                                   │            │                                   │
 └───────────────────────────────────┘            └───────────────────────────────────┘

  FAILOVER (manual)

  cluster1                                         cluster2
 ┌───────────────────────────────────┐            ┌───────────────────────────────────┐
 │                                   │            │                                   │
 │  ACM removes app                  │            │  New PVC ◄── latestImage snapshot │
 │  (Placement label removed)        │            │      │                            │
 │                                   │            │      ▼                            │
 │                                   │            │  ACM deploys MySQL                │
 │                                   │            │  MySQL starts with replicated data│
 │                                   │            │                                   │
 └───────────────────────────────────┘            └───────────────────────────────────┘
```

---

## Scope

Procedure to configure VolSync for asynchronous PVC replication between two
ACM-managed clusters and test a failover of a MySQL database from
**cluster1** to **cluster2**.

The application lifecycle is **managed by ACM** using an ApplicationSet
with a Placement. During failover, we change the Placement target (via
managed cluster labels) so ACM automatically removes the app from
`cluster1` and deploys it on `cluster2`.

The PVC (`mysql-pv-claim`) is **not** managed by ArgoCD: it is created
manually (see section 2.1). ArgoCD deploys the Deployment, Service,
and Secret only.

> **Important**: VolSync only supports PVCs with `volumeMode: Filesystem`
> (the default). Raw `volumeMode: Block` PVCs are not supported.

---

## Prerequisites

- ACM hub cluster (2.13+) with two managed clusters imported: `cluster1`
(source) and `cluster2` (destination).
- A CSI driver with **VolumeSnapshot** support on both clusters.
- Network connectivity from `cluster1` to `cluster2` (LoadBalancer or Submariner).
- OpenShift GitOps (ArgoCD) installed on the hub with ACM integration.
- `oc` CLI configured with hub cluster credentials. Extract managed cluster
kubeconfigs if needed:

```bash
CLUSTER_NAME=cluster1
SECRET_NAME=$(oc get secret -n $CLUSTER_NAME -o name | grep 'admin-kubeconfig')
oc extract $SECRET_NAME -n $CLUSTER_NAME --keys=kubeconfig --to=- > /tmp/${CLUSTER_NAME}-kubeconfig
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get nodes
```

```bash
CLUSTER_NAME=cluster2
SECRET_NAME=$(oc get secret -n $CLUSTER_NAME -o name | grep 'admin-kubeconfig')
oc extract $SECRET_NAME -n $CLUSTER_NAME --keys=kubeconfig --to=- > /tmp/${CLUSTER_NAME}-kubeconfig
oc --kubeconfig=/tmp/${CLUSTER_NAME}-kubeconfig get nodes
```

```bash
export cluster1_KUBECONFIG=/tmp/cluster1-kubeconfig
export cluster2_KUBECONFIG=/tmp/cluster2-kubeconfig
```

---

## 1. Install VolSync on both managed clusters

```bash
cat <<'EOF' | oc apply -f -
---
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: volsync
  namespace: cluster1
spec: {}
---
apiVersion: addon.open-cluster-management.io/v1alpha1
kind: ManagedClusterAddOn
metadata:
  name: volsync
  namespace: cluster2
spec: {}
EOF
```

Verify:

```bash
oc get ManagedClusterAddOn volsync -n cluster1
oc get ManagedClusterAddOn volsync -n cluster2
kubectl --kubeconfig=$cluster1_KUBECONFIG -n volsync-system get deployment volsync
kubectl --kubeconfig=$cluster2_KUBECONFIG -n volsync-system get deployment volsync
```

---

## 2. Deploy the Workload via ACM Application - ApplicationSet

Clone the Git repository:

```bash
cd /tmp
git clone https://github.com/CSA-RH/argodemo3_mysql-volsync.git
cd argodemo3_mysql-volsync
```

### 2.1 Create the namespace and PVC on cluster1

The PVC is **not** managed by ArgoCD (to prevent `selfHeal` conflicts
with VolSync). Create the namespace and the PVC manually:

```bash
oc --kubeconfig=$cluster1_KUBECONFIG create namespace mysql-volsync

cat <<'EOF' | oc --kubeconfig=$cluster1_KUBECONFIG apply -f -
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  namespace: mysql-volsync
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
EOF
```

### 2.2 Label cluster1 as the active target and make sure that Cluster2 doesnt have it set

```bash
oc label managedcluster cluster1 app-mysql-volsync=active --overwrite
oc label managedcluster cluster2 app-mysql-volsync- --overwrite
```

### 2.3 Create the Placement and ApplicationSet on the hub

```bash
oc apply -f argocd/placement.yaml
oc apply -f argocd/applicationset-push.yaml
```

### 2.4 Verify ACM deployed MySQL on cluster1

```bash
oc get applications.argoproj.io -n openshift-gitops | grep mysql-volsync
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync get pods
```

Wait for the MySQL pod to be `Ready`:

```bash
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync wait \
  --for=condition=available deployment/mysql-volsync --timeout=120s
```

### 2.5 Write test data to MySQL

Connect to MySQL and create test data that will prove DR replication.
The Red Hat MySQL image allows root to connect locally without a password:

```bash
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync exec deploy/mysql-volsync -- \
  mysql -u root -e "
    USE synced;
    CREATE TABLE IF NOT EXISTS demo (id INT AUTO_INCREMENT PRIMARY KEY, msg VARCHAR(255), created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP);
    INSERT INTO demo (msg) VALUES ('written on cluster1 before failover');
    SELECT * FROM demo;
  "
```

Verify the data was persisted:

```bash
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync exec deploy/mysql-volsync -- \
  mysql -u root -e "USE synced; SELECT * FROM demo;"
```

Expected output:

```
+----+------------------------------------+---------------------+
| id | msg                                | created_at          |
+----+------------------------------------+---------------------+
|  1 | written on cluster1 before failover | 2026-07-06 13:55:00 |
+----+------------------------------------+---------------------+
```

---

## 3. Identify storage and snapshot classes

```bash
oc --kubeconfig=$cluster1_KUBECONFIG get storageclass
oc --kubeconfig=$cluster2_KUBECONFIG get storageclass
oc --kubeconfig=$cluster1_KUBECONFIG get volumesnapshotclass
oc --kubeconfig=$cluster2_KUBECONFIG get volumesnapshotclass
```

Set variables (replace with your actual class names):

```bash
export SRC_STORAGE_CLASS="gp3-csi"
export DST_STORAGE_CLASS="gp3-csi"
export DST_SNAPSHOT_CLASS="csi-aws-vsc"
```

---

## 4. Create the ReplicationDestination on cluster2

```bash
oc --kubeconfig=$cluster2_KUBECONFIG create namespace mysql-volsync
```

```bash
cat <<EOF | oc --kubeconfig=$cluster2_KUBECONFIG apply -f -
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: mysql-data-dest
  namespace: mysql-volsync
spec:
  rsyncTLS:
    serviceType: LoadBalancer
    copyMethod: Snapshot
    capacity: 2Gi
    accessModes:
      - ReadWriteOnce
    storageClassName: ${DST_STORAGE_CLASS}
    volumeSnapshotClassName: ${DST_SNAPSHOT_CLASS}
EOF
```

Capture the connection info from the secondary cluster (cluster2). The
ReplicationDestination exposes a LoadBalancer address (the entry point
on cluster2 for storage replication, Submariner would be another
option) and generates a TLS-PSK secret (to authenticate the rsync-TLS
connection). Both are needed to configure the ReplicationSource on the
primary cluster (cluster1):

```bash
DEST_ADDRESS=$(oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync \
  get replicationdestination mysql-data-dest \
  -o jsonpath='{.status.rsyncTLS.address}')

DEST_SECRET=$(oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync \
  get replicationdestination mysql-data-dest \
  -o jsonpath='{.status.rsyncTLS.keySecret}')

echo "Destination address: $DEST_ADDRESS"
echo "Destination key secret: $DEST_SECRET"
```

---

## 5. Copy the TLS key secret to cluster1

Both clusters must share the same TLS-PSK key to establish the
rsync-TLS tunnel. The key was auto-generated on cluster2; copy it to
cluster1:

```bash
PSK_DATA=$(oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync \
  get secret $DEST_SECRET -o jsonpath='{.data.psk\.txt}')

cat <<EOF | oc --kubeconfig=$cluster1_KUBECONFIG apply -f -
---
apiVersion: v1
kind: Secret
metadata:
  name: ${DEST_SECRET}
  namespace: mysql-volsync
type: Opaque
data:
  psk.txt: ${PSK_DATA}
EOF
```

Verify:

```bash
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync get secret $DEST_SECRET
```

---

## 6. Create the ReplicationSource on cluster1

```bash
cat <<EOF | oc --kubeconfig=$cluster1_KUBECONFIG apply -f -
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: mysql-data-source
  namespace: mysql-volsync
spec:
  sourcePVC: mysql-pv-claim
  trigger:
    schedule: "*/3 * * * *"
  rsyncTLS:
    keySecret: ${DEST_SECRET}
    address: ${DEST_ADDRESS}
    copyMethod: Snapshot
    storageClassName: ${SRC_STORAGE_CLASS}
EOF
```

---

## 7. Verify replication is working

Wait for at least one sync cycle (3 minutes), then check:

**Source status (cluster1):**

```bash
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync \
  get replicationsource mysql-data-source \
  -o jsonpath='{.status.lastSyncTime}{"\n"}{.status.lastSyncDuration}{"\n"}{.status.nextSyncTime}'
```

**Destination status (cluster2):**

```bash
oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync \
  get replicationdestination mysql-data-dest \
  -o jsonpath='{.status.latestImage.name}{"\n"}{.status.lastSyncTime}'
```

**Verify snapshot exists on cluster2:**

```bash
oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync get volumesnapshot
```

### 7.1 Inspect VolSync objects on both clusters

After a successful sync cycle, VolSync creates temporary objects on the
source and persistent objects on the destination. Use the commands below
to inspect them.

> **Note:** Each cluster may be in a different AWS region. The commands
> below derive the region automatically from the cluster's
> `infrastructure/cluster` object.

Load your AWS credentials:

```bash
export AWS_ACCESS_KEY_ID="$(< ~/.passwords/aws_key_id)"
export AWS_SECRET_ACCESS_KEY="$(< ~/.passwords/aws_access_key)"
```

**Source cluster (cluster1), during a sync cycle:**

VolSync creates a temporary VolumeSnapshot and a temporary PVC (restored
from the snapshot) to read data from. Both are deleted after the sync
completes, so they are only visible while a sync is in progress.

```bash
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync get volumesnapshot
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync get pvc
```

To see the underlying EBS snapshots and volumes in AWS, filter by the
cluster infrastructure ID tag (`kubernetes.io/cluster/<infra-id>`):

```bash
INFRA_ID=$(oc --kubeconfig=$cluster1_KUBECONFIG \
  get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
REGION=$(oc --kubeconfig=$cluster1_KUBECONFIG \
  get infrastructure cluster -o jsonpath='{.status.platformStatus.aws.region}')

aws ec2 describe-snapshots --owner-ids self --region "$REGION" \
  --filters "Name=tag:kubernetes.io/cluster/$INFRA_ID,Values=owned" \
  --query 'Snapshots[].{ID:SnapshotId,State:State,Size:VolumeSize,Start:StartTime,Name:Tags[?Key==`Name`].Value|[0]}' \
  --output table
```

**Destination cluster (cluster2), persistent objects:**

The destination keeps the VolSync-managed PVC (receives synced data)
and the latest VolumeSnapshot (`latestImage`). These persist across
sync cycles.

```bash
oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync get volumesnapshot
oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync get pvc
```

```bash
INFRA_ID=$(oc --kubeconfig=$cluster2_KUBECONFIG \
  get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
REGION=$(oc --kubeconfig=$cluster2_KUBECONFIG \
  get infrastructure cluster -o jsonpath='{.status.platformStatus.aws.region}')

aws ec2 describe-snapshots --owner-ids self --region "$REGION" \
  --filters "Name=tag:kubernetes.io/cluster/$INFRA_ID,Values=owned" \
  --query 'Snapshots[].{ID:SnapshotId,State:State,Size:VolumeSize,Start:StartTime,Name:Tags[?Key==`Name`].Value|[0]}' \
  --output table
```

---

## 8. Failover: cluster1 to cluster2

Failover sequence:

1. (Optional) Quiesce Workload.
2. Create a PVC on `cluster2` from the latest VolSync snapshot.
3. Move the Placement target to `cluster2`.

### 8.1 (Optional) Quiesce Workload

If the stateful workload requires a specific procedure before being
stopped (e.g., flushing buffers, closing connections, draining queues),
perform it now, before triggering the final sync.

The scheduled sync runs every 3 minutes. Any data written after the
last successful sync and before the failover is **lost** (this gap is
the RPO). Triggering a final sync right before failover captures the
most recent state and reduces data loss.

If application-consistent failover is needed (e.g., for a database),
quiesce the workload **before** triggering the final sync. The
quiescence procedure is application-specific, always refer to the
vendor documentation for the correct steps.

### 8.2 Trigger a final sync and stop the old replication

Setting `trigger.manual` runs one last sync and then pauses the cron
schedule, preventing further syncs from cluster1 after failover.

```bash
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync patch replicationsource mysql-data-source \
  --type merge -p '{"spec":{"trigger":{"manual":"final-sync"}}}'
```

Wait for `lastSyncTime` to update:

```bash
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync \
  get replicationsource mysql-data-source -w
```

> The `trigger.manual` set above also stops the cron schedule, which
> prevents stale syncs from cluster1 after failover. To resume the
> schedule on failback, remove the `manual` field with:
> `oc patch replicationsource ... --type json -p '[{"op":"remove","path":"/spec/trigger/manual"}]'`

> VolSync does **not** automatically reverse the replication direction.
> The application on cluster2 is now **unprotected**, if cluster2
> fails, the data is lost. To protect it, you would need to manually
> set up reverse replication: create a new ReplicationDestination on
> cluster1, a new ReplicationSource on cluster2, and copy the TLS-PSK
> secret. With Regional DR (Ramen + ODF), the `DRPlacementControl`
> orchestrates this automatically, but with standalone VolSync + ACM
> policies as in this demo, it is entirely manual and out of scope.

### 8.3 Create the PVC on cluster2 from the latest snapshot

Create a PVC that references the `ReplicationDestination` directly. The
VolSync volume populator automatically selects the latest snapshot.
The PVC will stay in `Pending` until a pod tries to mount it
(`WaitForFirstConsumer`):

```bash
cat <<EOF | oc --kubeconfig=$cluster2_KUBECONFIG apply -f -
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  namespace: mysql-volsync
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: ${DST_STORAGE_CLASS}
  dataSourceRef:
    kind: ReplicationDestination
    apiGroup: volsync.backube
    name: mysql-data-dest
EOF
```

```bash
oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync get pvc
```

### 8.4 Move the Placement to cluster2

```bash
oc label managedcluster cluster1 app-mysql-volsync-
oc label managedcluster cluster2 app-mysql-volsync=active
```

ACM removes MySQL from `cluster1` and deploys it on `cluster2`. If
quiescence was used, the Deployment arrives with `replicas: 0` (no pods yet).


---

## 9. Verify the failover

**Check ACM moved the Application:**

```bash
oc get applications.argoproj.io -n openshift-gitops | grep mysql-volsync
```

**Check MySQL is running on cluster2:**

```bash
oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync wait \
  --for=condition=available deployment/mysql-volsync --timeout=120s
```

**Verify the replicated data survived failover:**

```bash
oc --kubeconfig=$cluster2_KUBECONFIG -n mysql-volsync exec deploy/mysql-volsync -- \
  mysql -u root -e "USE synced; SELECT * FROM demo;"
```

Expected output: the row `"written on cluster1 before failover"` should appear, proving the MySQL data was replicated by VolSync and survived the failover.

**Verify app is gone from cluster1:**

```bash
oc --kubeconfig=$cluster1_KUBECONFIG -n mysql-volsync get pods
```

---

## Cleanup

Delete ACM policies and the ApplicationSet **before** deleting the
namespaces, otherwise the enforce policies will recreate the
VolSync objects.

**On hub (first):**

```bash
oc delete policy volsync-mysql-replication-source -n acm-policies
oc delete policy volsync-mysql-replication-destination -n acm-policies
oc delete applicationset mysql-volsync -n openshift-gitops
oc delete placement mysql-volsync-placement -n openshift-gitops
oc label managedcluster cluster1 app-mysql-volsync-
oc label managedcluster cluster2 app-mysql-volsync-
oc --kubeconfig=$cluster1_KUBECONFIG delete namespace mysql-volsync
oc --kubeconfig=$cluster2_KUBECONFIG delete namespace mysql-volsync
```

Deleting the namespace cascades to all objects inside it. Specifically:

- **VolumeSnapshot** deletion triggers **VolumeSnapshotContent** deletion,
which deletes the underlying **EBS snapshot** in AWS.
- **PVC** deletion triggers **PV** deletion, which deletes the underlying
**EBS volume** in AWS.

Verify no orphaned EBS resources remain:

```bash
INFRA_ID=$(oc --kubeconfig=$cluster2_KUBECONFIG \
  get infrastructure cluster -o jsonpath='{.status.infrastructureName}')
REGION=$(oc --kubeconfig=$cluster2_KUBECONFIG \
  get infrastructure cluster -o jsonpath='{.status.platformStatus.aws.region}')

aws ec2 describe-snapshots --owner-ids self --region "$REGION" \
  --filters "Name=tag:kubernetes.io/cluster/$INFRA_ID,Values=owned" \
  --query 'Snapshots[].{ID:SnapshotId,Size:VolumeSize,Start:StartTime}' \
  --output table

aws ec2 describe-volumes --region "$REGION" \
  --filters "Name=tag:kubernetes.io/created-for/pvc/namespace,Values=mysql-volsync" \
  --query 'Volumes[].{ID:VolumeId,Size:Size,State:State}' \
  --output table
```

---

## Key considerations

- **PVC must not be in the ArgoCD-managed path**: the Deployment references
the PVC by name, but the PVC definition is not in Git. VolSync manages
the PVC lifecycle. If ArgoCD manages the PVC, `selfHeal` recreates it
empty on failover, causing data loss.
- **Restore PVC before moving Placement**: the PVC must exist on `cluster2`
before ACM deploys the app there, or pods will be stuck in `Pending`.
- **RPO** = the replication schedule interval (3 minutes). Data written
between the last sync and the failure is lost unless a final sync is
triggered.
- **Failover is NOT automatic**: VolSync only replicates data. You
orchestrate the failover by restoring the PVC and re-labelling the
clusters.
- **Quiescence**: This demo takes **crash-consistent** snapshots,
MySQL may be mid-transaction when the snapshot fires. On restore,
InnoDB runs crash recovery automatically. For **application-consistent**
backups (no crash recovery needed), VolSync offers two mechanisms:
  - **PVC copy triggers**: VolSync pauses at the snapshot step and waits
  for an external signal (annotation). A controller or object, e.g. CronJob or sidecar can
  quiesce the database, set the annotation, and release it after the
  snapshot completes.
  - **Manual triggers**: Replace `trigger.schedule` with `trigger.manual`.
  Each sync runs only when you change the trigger value, giving full
  control over when to quiesce and snapshot.
  ([docs](https://volsync.readthedocs.io/en/latest/usage/pvccopytriggers.html))
  ([discussion](https://github.com/backube/volsync/discussions/1414))

---

## References

- [ACM 2.16: Converting a replicated image to a usable PVC](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/business_continuity/index#converting-replicated-image)
- [ACM 2.16 Business Continuity: VolSync](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/business_continuity/index)
- [ACM 2.16: Converting a replicated image to a usable PVC](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/business_continuity/index#converting-replicated-image)

