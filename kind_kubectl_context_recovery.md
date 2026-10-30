# Fixing Kubernetes `localhost:8080` Connection Refused in a Kind Devcontainer

## Overview

This guide documents how to fix `kubectl` failing with:

```text
The connection to the server localhost:8080 was refused
```

In this case, the Kind cluster `rwml-68f3` had been created, but `kubectl` had no Kubernetes contexts configured. Without a valid context/kubeconfig, `kubectl` attempted to connect to `localhost:8080`.

## Environment

- Workspace: `/workspaces/Aladin`
- Working directory: `/workspaces/Aladin/deployments/dev/kind`
- Devcontainer shell user: `root`
- Kind cluster: `rwml-68f3`
- Expected Kubernetes context: `kind-rwml-68f3`

## Symptoms

Running:

```bash
kubectl get pods
```

returned errors similar to:

```text
Get "http://localhost:8080/api?timeout=32s": dial tcp [::1]:8080: connect: connection refused
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

Checking the contexts:

```bash
kubectl config get-contexts
kubectl config current-context
```

showed no configured contexts/current context.

## Resolution

### 1. Check whether the Kind cluster exists

```bash
kind get clusters
```

Confirm that `rwml-68f3` appears in the output.

### 2. Export the cluster's kubeconfig

```bash
kind export kubeconfig --name rwml-68f3
```

This writes/updates the Kubernetes connection details for the existing Kind cluster in the kubeconfig used by `kubectl`.

### 3. Confirm the context exists

```bash
kubectl config get-contexts
```

Expected context:

```text
kind-rwml-68f3
```

### 4. Select the correct context

```bash
kubectl config use-context kind-rwml-68f3
```

### 5. Verify the Kubernetes API connection

```bash
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
```

The commands should now communicate with the Kind cluster. Check that the node and pods are in the expected states.

### 6. Open K9s

```bash
k9s --context kind-rwml-68f3 --all-namespaces
```

This opens K9s using the correct cluster context and displays resources across namespaces.

## Quick Recovery Commands

If the cluster still exists but the kubeconfig context is missing, run:

```bash
kind get clusters
kind export kubeconfig --name rwml-68f3
kubectl config use-context kind-rwml-68f3
kubectl get nodes
kubectl get pods -A
```

If the context is still missing, inspect the environment and kubeconfig directory:

```bash
echo "$KUBECONFIG"
ls -la ~/.kube/
kubectl config get-contexts
```

If `KUBECONFIG` points to an incorrect file, you can test with the default kubeconfig in the current terminal session:

```bash
unset KUBECONFIG
kind export kubeconfig --name rwml-68f3
kubectl config use-context kind-rwml-68f3
kubectl get nodes
```

Only unset `KUBECONFIG` if the custom path is incorrect or you intend to use the default `~/.kube/config`.

## Related Docker Network Warning

During cluster setup, Docker also reported:

```text
Error response from daemon: invalid pool request: Pool overlaps with other one on this address space
```

This indicates a requested Docker network subnet overlaps with an existing network. The Kind cluster subsequently reported that it had been created, so the Kubernetes context issue was addressed first.

If the Docker network error occurs again, inspect networks before making changes:

```bash
docker network ls
docker network inspect <network-name>
```

Do not delete Docker networks indiscriminately: other containers or development services may depend on them. Choose a non-overlapping subnet only after checking existing network configurations and the Kind setup script.

## Notes

- A missing Kubernetes context is a kubeconfig/configuration problem; it does not automatically mean the cluster must be recreated.
- Do not delete and recreate the cluster as the first troubleshooting step.
- Run `kubectl get nodes` before opening K9s to verify connectivity.
- The kubeconfig is environment-specific. If the devcontainer is recreated or a different shell/user is used, verify the context again.
