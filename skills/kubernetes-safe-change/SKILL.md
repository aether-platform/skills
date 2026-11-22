---
name: kubernetes-safe-change
description: Make and verify changes to a Kubernetes cluster without hitting the wrong cluster or leaving it in an unknown state, using kubectl, kustomize and helm. Use when you must find and prepare the right kubeconfig context, inspect a cluster read-only, check whether a manifest or Helm values change will actually roll out pods, validate it with server-side dry-run and admission, apply it and watch the rollout, give automation a least-privilege ServiceAccount and prove its RBAC, roll back, or run a disposable kind/k3d test cluster and clean it up.
---

# Kubernetes safe change

This skill gives a fixed procedure for working on a Kubernetes cluster as an
agent: identify the cluster, look before writing, predict what a change will
do, validate it on the API server, apply it, verify it, and keep the way back.
It uses `kubectl` (with its built-in kustomize), `helm` and, for throwaway
clusters, `kind` or `k3d`. It does not add any tools of its own.

Run `kubectl <command> --help` and `helm <command> --help` for the exact
options of the versions you have; flag names differ between major versions
(for example Helm 4 uses `--rollback-on-failure` where Helm 3 used
`--atomic`).

## Ground rules

- **Know which cluster you are talking to.** Never rely on whatever
  `~/.kube/config` or the current context happens to be. Use a kubeconfig file
  made for the task and prove which cluster it reaches (step A2) before any
  command that writes.
- **Read-only until a write is approved.** Inspection, rendering, `diff` and
  server-side dry-run are free. `apply`, `patch`, `delete`, `scale`,
  `rollout restart`, `helm upgrade/rollback/uninstall` need the owner's go for
  that cluster and that change.
- **Know what your credential can do.** A kubeconfig that came from a cluster
  installer is usually cluster-admin. Say so in the record. If the task only
  needs to read, ask for (or create, with approval) a read-only identity.
- **Merged is not applied.** A change merged into an infrastructure
  repository reaches a cluster only if something syncs it (a GitOps controller
  watching that repository and path) or someone applies it. Check which one
  holds before saying a change is live.
- **Secrets stay secret.** Do not print Secret data, tokens or certificates to
  the terminal, logs, commits or reports. When a value must be moved, write it
  straight to a file with mode 0600 and record only the file name.
- **Touch only your own resources.** On a shared machine or cluster, other
  clusters, namespaces and releases belong to someone else. Do not stop,
  delete or reconfigure them, even to free resources.
- **No internal values in shared places.** Do not paste API server addresses,
  node IPs, cluster names or kubeconfig contents into public repositories,
  issues or examples.

## A. Find and prepare the context

### A1. Make a task kubeconfig

1. Copy the kubeconfig you were given into the work folder (directory 0700,
   file 0600) and point only this session at it:

   ```bash
   export KUBECONFIG=<work folder>/kube/<cluster>.yaml
   kubectl config get-contexts
   ```

2. If the API server is only reachable through a bastion, open a tunnel for
   the session (`ssh -f -N -L 127.0.0.1:<port>:<api host>:<port> <bastion>`)
   and change only the `server:` line of the task copy — and only if the
   server certificate is valid for the new address. Close the tunnel when you
   are done.
3. Nothing goes into `~/.kube/config` or the home directory.

### A2. Prove which cluster it is

Use read-only signals that only the intended cluster has, not just its name:

```bash
kubectl cluster-info
kubectl get nodes -o wide
kubectl auth whoami
kubectl get ns
kubectl get crd | grep -i <expected operator>
```

- The namespaces, operators and releases should match what the
  infrastructure repository says this cluster runs. Write down why you
  concluded it is the right one.
- A cluster whose API name resolves to `127.0.0.1` through a provider's
  wildcard DNS, or that has none of the expected namespaces, is not the one —
  stop and ask.
- Note the client/server version skew `kubectl version` warns about.

## B. Look before you change

```bash
kubectl -n <ns> get deploy,sts,ds,svc,ingress,pvc,cm,secret
kubectl -n <ns> describe <kind>/<name>
kubectl -n <ns> events --for <kind>/<name>
kubectl get storageclass
helm -n <ns> list
helm -n <ns> history <release>
helm -n <ns> get values <release>
```

Check in particular:

- **How the workload got there.** A Helm release, a kustomize apply, or a
  GitOps controller (for Flux: `kubectl get gitrepositories,kustomizations,helmreleases -A`).
  Change it through the same path, or the next sync will undo yours.
- **StorageClass defaults.** If more than one StorageClass is marked
  `(default)`, which one a new PVC gets is not defined. Set
  `storageClassName` explicitly in new PVCs.
- **Things already broken** (ImagePullBackOff, releases not Ready). List them
  as pre-existing so they are not blamed on your change.

## C. Predict what the change does

### C1. Render and compare

Render the current and the changed version and diff them, before touching the
cluster:

```bash
kubectl kustomize <dir> > after.yaml            # kustomize
helm template <release> <chart> -f values.yaml > after.yaml   # helm
```

- Look at which **Pod templates** change. Only a change to a Pod template
  (`spec.template`) starts a rollout of that Deployment / StatefulSet /
  DaemonSet.
- **A ConfigMap or Secret change alone does not restart Pods.** Files mounted
  with `subPath`, values read into environment variables, and files that an
  init container renders at start-up stay as they were until the Pod is
  recreated. If the change must take effect, the chart needs a
  `checksum/config`-style annotation on the Pod template for the data that
  Pod uses (hashing only the keys it consumes avoids restarting every
  component on every change), or the owner must approve a restart.
- Before relying on a "reload" command of the application, check that it
  re-reads the file you changed and that the file in the Pod actually
  changed.

### C2. Validate on the API server

```bash
kubectl diff -k <dir>                          # or -f after.yaml
kubectl apply -k <dir> --dry-run=server
helm upgrade <release> <chart> -n <ns> -f values.yaml --dry-run=server
```

- Server-side dry-run runs admission (Pod Security Admission, validating
  webhooks, quota) without persisting anything. A Pod spec that a namespace's
  `pod-security.kubernetes.io/enforce` level rejects fails here.
- `helm ... --dry-run` prints the rendered manifests, including Secrets. Use
  `--hide-secret` and do not keep the output in shared places.
- A very large object (for example a ConfigMap of a few hundred KB) can exceed
  the size limit of the annotation that client-side apply writes. Apply it
  with `--server-side`.

## D. Apply and roll out

1. Save what you are about to replace (`kubectl get <kind>/<name> -o yaml`
   into the work folder, `helm get values` / `helm history`).
2. Apply through the path found in B:

   ```bash
   kubectl apply -k <dir>                       # add --server-side if needed
   helm upgrade --install <release> <chart> -n <ns> -f values.yaml \
     --wait --rollback-on-failure --timeout 10m
   ```

3. Watch it finish:

   ```bash
   kubectl -n <ns> rollout status deploy/<name> --timeout=5m
   kubectl -n <ns> wait --for=condition=Ready pod -l <selector> --timeout=5m
   ```

4. Verify the service the way its users reach it (its URL or Service, a
   health endpoint), not only the Pod status.

### Rollback

```bash
kubectl -n <ns> rollout history deploy/<name>
kubectl -n <ns> rollout undo deploy/<name> [--to-revision=<n>]
helm -n <ns> rollback <release> <revision> --wait
```

Write the rollback command, with the concrete revision, into the record
before you apply.

## E. Least privilege for automation

When a CI job or another workload has to change something in the cluster:

1. Give it its own ServiceAccount and a namespaced Role with only the verbs
   and resources it needs, narrowed with `resourceNames` where possible (for
   example `get` and `patch` on one Deployment, nothing on `pods/exec` or
   `secrets`).
2. Workloads that do not call the API get
   `automountServiceAccountToken: false`.
3. Prove the RBAC, both what is allowed and what is not:

   ```bash
   SA=system:serviceaccount:<ns>:<sa>
   kubectl auth can-i patch deployment/<name> -n <ns> --as=$SA     # yes
   kubectl auth can-i create pods/exec -n <ns> --as=$SA            # no
   kubectl auth can-i get secrets -n <ns> --as=$SA                 # no
   kubectl auth can-i patch deployments -n default --as=$SA        # no
   ```

4. Prefer a short-lived token (`kubectl create token <sa> --duration=...`)
   over a long-lived token Secret. Hand it over as a file, never in a command
   line or log.

## F. Disposable test clusters (kind / k3d)

Use a throwaway local cluster to try a change before it goes near a shared
cluster:

1. **Check room first.** `df -h` on the disk that holds the container
   runtime's data. Building and rebuilding clusters fills it quickly; when
   the disk gets tight, kubelet evicts Pods and every result after that point
   is invalid.
2. Create a cluster with its own name for this task
   (`kind create cluster --name <task>` or `k3d cluster create <task>`) and
   leave other clusters on the machine alone.
3. Pin what you install: image tags with digests, release versions of
   operators and CLIs. Record them in a file next to the scripts.
4. Distribution differences are findings, not noise. A k3s-based cluster, for
   example, keeps the CNI binaries and configuration under its own paths, so
   add-ons that assume the defaults need those paths set explicitly. Write
   such adjustments down; they do not prove anything about a managed cluster.
5. Stop conditions: disk pressure, Pods evicted, a node that reports Ready but
   whose Pods cannot be reached (`kubectl exec` fails). Stop, record, and do
   not report results taken after that point.
6. Tear down only what you created (`kind delete cluster --name <task>` /
   `k3d cluster delete <task>` and the networks you added), then check that
   the other clusters are still running and the disk is back where it was.

## Report

```
Cluster:     <how identified (A2), not its address>   credential: <admin / scoped / read-only>
Change:      <what, through which path: helm / kustomize / GitOps>
Predicted:   Pod templates changed: <list>   rollout expected: <yes/no>
Validated:   diff=<summary>  server dry-run=<ok/denied: reason>
Applied:     <time>  rollout status=<complete>  service check=<url/endpoint: result>
Rollback:    <exact command with revision>
Pre-existing problems: <list>
Not done / needs approval: <list>
```
