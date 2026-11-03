# Lab 3 — Running a Pod in Kubernetes

## Lesson concepts: Kubernetes, Containers & Cloud

### CRI — Container Runtime Interface

The CRI is the component that makes it possible to run containers on servers. It sits between Kubernetes and the actual container engine (like containerd or CRI-O), so Kubernetes doesn't need to know the details of *how* containers run — it just talks to the CRI and the CRI handles the rest.

### Minikube

A local Kubernetes cluster you can install on any platform (Linux, macOS, Windows). It lets you practice Kubernetes commands without needing a real cloud cluster.

### Decoupling in cloud environments

A core cloud-native principle: separating concerns so components can change independently. Examples:
- Pods decouple your app from the underlying server
- ConfigMaps decouple configuration from container images
- Services decouple network access from individual pod IPs

### Pod

The minimal unit that Kubernetes manages. A pod wraps one or more containers and is the basic building block for orchestrating containers in the cloud. Kubernetes doesn't manage containers directly — it manages pods.

### ConfigMap

Stores configuration data in a decoupled way, separate from the container image. This means you can change config (env vars, settings files) without rebuilding the image.

### Services & Ingress

- **Service**: exposes a pod (or group of pods) on the network, with a stable IP/DNS name even as pods come and go
- **Ingress**: routes external HTTP/HTTPS traffic into services inside the cluster (acts like a reverse proxy / load balancer at the edge)

---

## Errors & How I Fixed Them

### ❌ Error 1 — Grade script not found in PATH

**Command run:**
```bash
lab3-grade.sh
```

**Error:**
```
bash: lab3-grade.sh: command not found
```

**Root cause:** The script was in the current directory but not in `$PATH`, so the shell couldn't find it by name alone.

**✅ Fix — Prefix with `./` to run from the current directory:**
```bash
./lab3-grade.sh
```

---

### ❌ Error 2 — Wrong `kubectl delete` syntax

**Commands tried:**
```bash
kubectl delete myfirstapp
kubectl delete nginx
```

**Error:**
```
error: the server doesn't have a resource type "myfirstapp"
error: the server doesn't have a resource type "nginx"
```

**Root cause:** `kubectl delete` needs the *resource type* first, then the name. Without it, kubectl tries to interpret the name as a resource type, which doesn't exist.

**✅ Fix — Specify the resource type (`pod`) before the name:**
```bash
kubectl delete pod myfirstapp
```

---

### ❌ Error 3 — Wrong `kubectl get` syntax

**Command run:**
```bash
kubectl get pod all
```

**Error:**
```
Error from server (NotFound): pods "all" not found
```

**Root cause:** `all` here was treated as a pod *name*, not a keyword. `get pod all` means "get the pod named all".

**✅ Fix — Use `kubectl get pods` (plural) or `kubectl get all`:**
```bash
kubectl get pods        # list all pods
kubectl get all         # list all resources in the namespace
```

---

### ❌ Error 4 — Grade failed: pod named wrong

After deleting `myfirstapp`, the grade script expected a pod specifically named `lab3`.

**✅ Fix — Create the pod with the exact name the grade script checks for:**
```bash
kubectl run lab3 --image=nginx
```

---

## What I Learned

| Concept | Notes |
|---|---|
| `kubectl run <name> --image=<image>` | Creates and starts a pod with the given name and image |
| `kubectl get pods` | Lists pods in the current namespace |
| `kubectl delete pod <name>` | Deletes a specific pod — resource type must come before the name |
| `./script.sh` vs `script.sh` | Scripts in the current directory need `./` prefix unless their folder is in `$PATH` |
| Pod naming matters | The grade script checks for a pod by an exact name — always use the name the lab specifies |
| `kubectl get all` | Lists all resource types in the namespace (not just pods) |

---

## Commands Reference

| Command | What it does |
|---|---|
| `kubectl run lab3 --image=nginx` | Create a pod named `lab3` running the nginx image |
| `kubectl get pods` | List all running pods |
| `kubectl get all` | List all resources in the default namespace |
| `kubectl delete pod <name>` | Delete a pod by name |
| `kubectl describe pod <name>` | Show detailed info about a pod |
| `kubectl logs <name>` | Print the logs of a pod's container |

---

## Result

```
[OK]   Minikube is running
[OK]   Pod lab3 is running

[CONGRATS] You have successfully completed this lab!
```
