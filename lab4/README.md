# Lab 4 — Kubernetes Deployments & MariaDB

## What this lab was about

Generate a Kubernetes Deployment manifest using `kubectl create deployment --dry-run=client -o yaml`, deploy **two MariaDB pods**, troubleshoot failed starts using `kubectl describe` and `kubectl logs`, and use the lab grading script to check the result.

**Expected command to generate the manifest:**
```bash
kubectl create deployment newdb --image=mariadb --replicas=2 --dry-run=client -o yaml > newdb.yaml
kubectl apply -f newdb.yaml
```

**Example of a corrected manifest for this learning exercise** (not a copy of the original generated file):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: newdb
spec:
  replicas: 2
  selector:
    matchLabels:
      app: newdb
  template:
    metadata:
      labels:
        app: newdb
    spec:
      containers:
        - name: mariadb
          image: mariadb
          env:
            - name: MARIADB_ROOT_PASSWORD
              value: "password"
```

> **Learning-lab only:** The plaintext password above mirrors the local troubleshooting session. For actual deployments, use Kubernetes Secrets, persistent storage, and separate database instances or a properly designed replication setup. Two replicas of a standalone MariaDB Deployment do **not** automatically replicate data.

---

## Lesson concepts: Deployments, ReplicaSets & Pods

### Deployment

A Deployment declares the desired state of an application (for example, **2 replicas**). Kubernetes maintains that state by creating and updating ReplicaSets.

### ReplicaSet

A ReplicaSet makes sure the specified number of pods exist. When an old pod is deleted, it creates a replacement. Updating a Deployment creates a new ReplicaSet, while old ones may remain with zero desired replicas.

### Declarative YAML vs imperative commands

- `--dry-run=client -o yaml` generates a manifest without submitting it to the cluster.
- `kubectl apply -f newdb.yaml` creates or updates a resource using the manifest.
- `kubectl set env deploy newdb ...` updates the **live Deployment**, but does not rewrite the local `newdb.yaml`.

### Troubleshooting states

- **ErrImagePull / ImagePullBackOff:** Kubernetes cannot fetch the requested container image.
- **Error / repeated restarts:** The container started but exited; inspect the logs.
- **2/2 READY:** Both desired replicas are ready at that moment.

---

## Errors & How I Fixed Them

### ❌ Error 1 — Typo in the MariaDB image name

**Command run:**
```bash
kubectl create deploy newdb --image=maridb --replicas=2 --dry-run=client -o yaml > newdb.yaml
kubectl apply -f newdb.yaml
```

**Observed result:**
```text
0/2   ErrImagePull
Failed to pull image "maridb"
failed to resolve reference "docker.io/library/maridb:latest"
pull access denied, repository does not exist
```

**Root cause:** I wrote `maridb` instead of `mariadb`. Since the repository name was wrong, Kubernetes could not pull the image.

**✅ Fix — Correct the image in the manifest and apply it again:**
```bash
# Edit newdb.yaml: replace image: maridb with image: mariadb
nano newdb.yaml
kubectl apply -f newdb.yaml
```

Alternatively, regenerate the manifest with `--image=mariadb` if you do not need to preserve edits.

---

### ❌ Error 2 — Confusing resource deletion with pod deletion

**Commands tried:**
```bash
kubectl delete all
kubectl delete --all
kubectl delete pods --all
```

**Errors from the first two commands:**
```text
error: resource(s) were provided, but no name was specified
error: at least one resource must be specified to use a selector
```

**Root cause:** `kubectl delete` requires a resource type and either a resource name or `--all`. The third command did delete the pods, but the existing Deployments recreated them immediately, as shown by the new pod names and short ages.

**✅ Fix — Choose the resource I actually want to remove:**
```bash
kubectl delete pods --all              # deletes pods; Deployments replace them
kubectl delete deployment myweb        # deletes the Deployment and owned pods
kubectl delete deployment mydb
```

> In this session, deleting the `myweb` and `mydb` Deployments removed their managed pods. The default `kubernetes` Service remained, which is normal.

---

### ❌ Error 3 — MariaDB required an initialization password

After correcting the image, the pods started but exited.

**Log message:**
```text
[ERROR] [Entrypoint]: Database is uninitialized and password option is not specified
You need to specify one of MARIADB_ROOT_PASSWORD,
MARIADB_ROOT_PASSWORD_HASH, MARIADB_ALLOW_EMPTY_ROOT_PASSWORD
and MARIADB_RANDOM_ROOT_PASSWORD
```

**First attempt (not enough):**
```bash
kubectl set env deploy newdb MARIADB_PASSWORD=password
```

**Root cause:** `MARIADB_PASSWORD` does not satisfy the root-password requirement for initializing a new MariaDB instance. The container's own error message explicitly lists the required root-password options.

**✅ Fix — Supply the root-password variable required by this image:**
```bash
kubectl set env deploy newdb MARIADB_ROOT_PASSWORD=password
kubectl get deployment newdb
kubectl get pods -l app=newdb
```

**Observed result:** The Deployment reached **2/2 READY**. Its logs subsequently showed database initialization and `ready for connections` on port `3306`.

> The lab script's hint refers to `MYSQL_ROOT_PASSWORD`. The actual MariaDB image in this session requested `MARIADB_ROOT_PASSWORD`, which solved the observed error.

---

### ❌ Error 4 — Incorrect `kubectl logs` arguments

**Commands tried:**
```bash
kubectl logs
kubectl logs all
kubectl logs newdb-6fb6f7c5c8-9
```

**Errors:**
```text
POD or TYPE/NAME is a required argument for the logs command
pods "all" not found
pods "newdb-6fb6f7c5c8-9" not found
```

**Root cause:** The command requires a real pod name or supported resource reference. `all` is not a shortcut for collecting logs from every pod, and an abbreviated pod name is not enough.

**✅ Fix — List the pods, then supply a complete pod name:**
```bash
kubectl get pods
kubectl logs newdb-6fb6f7c5c8-9jtgv
kubectl describe pod newdb-6fb6f7c5c8-9jtgv
```

Pod names above are from this session and will change when pods are recreated.

---

### ❌ Error 5 — The grading script could not find `newdb.yaml`

**Command run:**
```bash
sudo ./labs/lab4-grade.sh
```

**Error:**
```text
[FAIL] I cannot find newdb.yaml anywhere in your home directory.
```

The file clearly existed under `~/learn/kubernetes/kubestep/newdb.yaml`.

**Root cause discovered by inspecting the script:**
```bash
FINDOUT=$(find ~ -name "newdb.yaml")
```

Running the script under `sudo` commonly expands `~` to `/root`, not the regular user's home folder. Running it without `sudo` made `find` recurse through unrelated Podman storage under `~/.local/share/containers/storage`, resulting in multiple `Permission denied` messages and a long-running search that was interrupted.

**✅ Fix — Prefer a narrow search in the local grading script:**
```bash
# In labs/lab4-grade.sh, replace the original FINDOUT line:
LAB_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
FINDOUT=$(find "$LAB_ROOT" -name "newdb.yaml" -print -quit 2>/dev/null)
```

Then run the script normally:
```bash
./labs/lab4-grade.sh
```

If the original script must remain unchanged, an alternative is:
```bash
sudo env HOME="$HOME" KUBECONFIG="$HOME/.kube/config" ./labs/lab4-grade.sh
```

That alternative addresses the home-directory mismatch, but the wide `find` may still take time. Note that the grading script also creates `/tmp/lab4check1` and `/tmp/lab4check2` state files; mixed root/non-root execution can cause permissions problems with these markers.

**Important:** The supplied terminal transcript does **not** show a successful grading run, so the grading-script problem remains unverified as resolved.

---

## What I Learned

| Concept | Notes |
|---|---|
| `kubectl create deploy --dry-run=client -o yaml` | Generate a reusable Deployment manifest without creating the resource yet |
| `kubectl apply -f` | Create or update resources from YAML |
| Desired replicas | Deployments and ReplicaSets keep pods running; deleting pods alone does not stop the Deployment |
| `ErrImagePull` | Inspect image spelling, registry and tag using `kubectl describe pod` |
| Container crashes | Use `kubectl logs POD` to find initialization and application errors |
| MariaDB initialization | A new database needs one of the accepted root-password options |
| `kubectl set env` | Changes the live Deployment and causes a rollout, but does not update the source YAML |
| `kubectl get all` vs `kubectl delete all` | `get all` lists resources; to delete, explicitly select a resource type/name or `--all` |
| Grading scripts and `sudo` | Running as root may change home-directory expansion, configuration paths, and file permissions |
| Database replicas | Two separate MariaDB pods are not a replicated database cluster |

---

## Commands Reference

| Command | What it does |
|---|---|
| `kubectl create deploy newdb --image=mariadb --replicas=2 --dry-run=client -o yaml > newdb.yaml` | Generate the two-replica Deployment YAML |
| `kubectl apply -f newdb.yaml` | Create or update the Deployment |
| `kubectl get all` | Inspect workload and Service resources in the namespace |
| `kubectl get pods -l app=newdb` | Show the pods belonging to this app |
| `kubectl get deployment newdb` | Check desired, ready and available replicas |
| `kubectl describe pods -l app=newdb` | Inspect status and events, including image pull failures |
| `kubectl logs POD_NAME` | Read container logs for a particular pod |
| `kubectl set env deploy newdb MARIADB_ROOT_PASSWORD=password` | Configure MariaDB's root password in the live Deployment (demo only) |
| `kubectl delete pods --all` | Delete pods, which Deployments will replace |
| `kubectl delete deployment newdb` | Delete the Deployment and its managed pods |
| `./labs/lab4-grade.sh` | Execute the local lab grader |

---

## Result

**✅ Kubernetes deployment working (observed):**

```text
NAME                    READY   UP-TO-DATE   AVAILABLE
deployment.apps/newdb   2/2     2            2

pod/newdb-6fb6f7c5c8-9jtgv   1/1   Running   0
pod/newdb-6fb6f7c5c8-p2rfn   1/1   Running   0
```

MariaDB's startup logs included:
```text
Database files initialized
MariaDB init process done. Ready for start up.
Server socket created on IP: '0.0.0.0', port: '3306'.
mariadbd: ready for connections.
```

**⚠️ Grading status:** Not confirmed. The most recent recorded grading attempt failed to locate `newdb.yaml` under `sudo`; a non-root attempt was interrupted while traversing protected Podman storage. Re-run the grader after fixing its search scope, and record the actual final output when available.
