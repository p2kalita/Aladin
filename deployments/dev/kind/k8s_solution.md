# Aladin Kubernetes + k9s Setup --- Solutions Log

This document records the steps that worked to open the Aladin
development environment, recover the Kubernetes configuration, verify
the Kind cluster, and display all namespaces in `k9s`.

## 1. Open the Aladin Dev Container terminal

1.  Open the Aladin project in VS Code.
2.  Reopen or attach to the project's Dev Container if it is not already
    running.
3.  Open a Bash terminal inside the Dev Container.
4.  Navigate to the Kind deployment directory:

``` bash
cd /workspaces/Aladin/deployments/dev/kind
```

> Run the Kubernetes commands below from a terminal where Docker,
> `kind`, `kubectl`, and `k9s` are available. The recorded environment
> used a Linux Bash shell inside the Dev Container.

## 2. Install `kubectl` if needed

During setup, the downloaded `kubectl` binary was installed with:

``` bash
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

This assumes the `kubectl` binary is already in the current directory
and the shell has permission to write to `/usr/local/bin`.

Verify the tools are available:

``` bash
kubectl version --client
kind version
k9s version
```

If a command is not found, install or locate that tool before
continuing.

## 3. Create or check the Kind cluster

The cluster name used by this project is `rwml-68f3`.

From the Kind deployment directory, the setup script was run:

``` bash
cd /workspaces/Aladin/deployments/dev/kind
bash create_cluster.sh
```

During this run, the script printed warnings/errors about:

-   The `rwml-68f3-network` network not being found during deletion.
-   A Docker network IP pool overlapping with another network.

Despite those messages, Kind reported that it created the cluster and
set the context to `kind-rwml-68f3`.

**Important:** Because the cluster creation reported network errors, do
not assume every part of the script completed cleanly. In this case, the
cluster was subsequently found and its control-plane node was healthy.
Avoid repeatedly recreating the cluster just to fix the earlier
warnings.

Check whether the cluster exists:

``` bash
kind get clusters
```

Expected cluster entry:

``` text
rwml-68f3
```

## 4. Fix an empty or missing kubeconfig

### Problem

Initially, `kubectl` had no usable context. The kubeconfig view showed
no clusters, contexts, or users, and `kubectl` attempted to connect to
`http://localhost:8080`.

### Solution that worked

Create the kubeconfig directory:

``` bash
mkdir -p ~/.kube
```

Export the kubeconfig for the existing Kind cluster:

``` bash
kind export kubeconfig --name rwml-68f3 --kubeconfig ~/.kube/config
```

Set the kubeconfig path for the current shell:

``` bash
export KUBECONFIG="$HOME/.kube/config"
```

List the contexts:

``` bash
kubectl config get-contexts
```

Select the Kind context:

``` bash
kubectl config use-context kind-rwml-68f3
```

Verify the selected context:

``` bash
kubectl config current-context
```

Expected output:

``` text
kind-rwml-68f3
```

### Note about new terminals

The `export KUBECONFIG=...` command applies to the current shell only.
However, `~/.kube/config` is kubectl's default configuration location,
so new shells will normally use it automatically as long as the file
persists in the same Dev Container.

If the Dev Container is rebuilt or the kubeconfig is lost, repeat the
export step. If a new terminal does not find the context, check:

``` bash
ls -l ~/.kube/config
kubectl config get-contexts
```

If necessary, re-export the config with the `kind export kubeconfig`
command above.

## 5. Verify the cluster and its nodes

Run:

``` bash
kubectl --context kind-rwml-68f3 get nodes
```

The setup was verified with a node similar to:

``` text
NAME                      STATUS   ROLES           VERSION
rwml-68f3-control-plane   Ready    control-plane   v1.31.4
```

The key success condition is that the node's status is `Ready`.

You can also run:

``` bash
kubectl --context kind-rwml-68f3 cluster-info
```

## 6. Verify Kubernetes system Pods

Run:

``` bash
kubectl --context kind-rwml-68f3 get pods -A
```

The following core Pods were observed running:

-   CoreDNS Pods in `kube-system`
-   The Kind control-plane etcd Pod
-   The `kindnet` Pod
-   The Kubernetes API server Pod
-   The controller manager Pod
-   The kube-proxy Pod
-   The scheduler Pod
-   The local-path storage provisioner Pod in `local-path-storage`

The observed Pods were `Running`, with `1/1` containers ready and zero
restarts at that time.

Use this command again whenever you want to check the cluster:

``` bash
kubectl --context kind-rwml-68f3 get pods -A
```

## 7. Open k9s across all namespaces

### Initial issue

When `k9s` was opened without selecting all namespaces, it showed a
message like:

``` text
No resources found for v1/pods in "default" namespace
```

That was not a cluster failure. It meant there were no Pods visible in
the currently selected `default` namespace; the Kubernetes system Pods
were in other namespaces.

### Command that worked

Open a Bash terminal and run:

``` bash
k9s --context kind-rwml-68f3 --all-namespaces
```

This explicitly selects the Kind context and starts the dashboard with
all namespaces included. This was the command that successfully
displayed the Pods in the dashboard.

It is fine to keep `k9s` running in one terminal and use `kubectl` in
another terminal. Both can inspect the same cluster when they use the
same kubeconfig and context.

## 8. Useful k9s navigation

While k9s is open:

  Key / command    Purpose
  ---------------- --------------------------------------------------------------
  `0`              Show all namespaces (in versions that support this shortcut)
  `:pods`          Open the Pods view
  `:deployments`   Open the Deployments view
  `:services`      Open the Services view
  `:nodes`         Open the Nodes view
  `Esc`            Go back to the previous view
  `q`              Quit k9s

Shortcuts can vary slightly by k9s version. If a shortcut does not work,
use the command prompt (usually `:`) and enter the resource name.

## 9. Starting it again later

When the Dev Container and Docker environment are available:

1.  Open a Bash terminal in the Dev Container.

2.  If needed, confirm the cluster exists:

    ``` bash
    kind get clusters
    ```

3.  Check the context:

    ``` bash
    kubectl config current-context
    ```

4.  If the context is missing, restore the kubeconfig:

    ``` bash
    mkdir -p ~/.kube
    kind export kubeconfig --name rwml-68f3 --kubeconfig ~/.kube/config
    export KUBECONFIG="$HOME/.kube/config"
    kubectl config use-context kind-rwml-68f3
    ```

5.  Open the dashboard:

    ``` bash
    k9s --context kind-rwml-68f3 --all-namespaces
    ```

### What closing things means

-   **Quit k9s:** Only the terminal dashboard closes; this does not
    delete the cluster.
-   **Close a terminal:** The command running in that terminal exits,
    but that alone does not necessarily stop the Kind containers.
-   **Stop or close the Dev Container:** The cluster may remain
    available if its Docker containers are still running. This depends
    on how the Dev Container and Docker are stopped.
-   **Shut down the PC:** Docker containers stop with the machine. After
    restarting, start Docker and reopen the Dev Container, then check
    the cluster and context before launching k9s.
-   **Rebuild or remove the Dev Container:** Files stored only inside
    that container, including `~/.kube/config`, may not persist.
    Re-export the kubeconfig if needed.

## 10. What we deliberately did not do

No Nginx or other test application was deployed. The goal was only to
restore access to the existing Kind cluster and make its current
resources visible in k9s.

## Quick recovery checklist

Copy and run these commands in a Bash terminal inside the Dev Container:

``` bash
cd /workspaces/Aladin/deployments/dev/kind

# Confirm the cluster exists
kind get clusters

# Restore kubeconfig if the context/config is missing
mkdir -p ~/.kube
kind export kubeconfig --name rwml-68f3 --kubeconfig ~/.kube/config
export KUBECONFIG="$HOME/.kube/config"

# Select and verify the context
kubectl config use-context kind-rwml-68f3
kubectl config current-context

# Verify cluster health and Pods
kubectl --context kind-rwml-68f3 get nodes
kubectl --context kind-rwml-68f3 get pods -A

# Open k9s across all namespaces
k9s --context kind-rwml-68f3 --all-namespaces
```

**Expected end state:** the `rwml-68f3` Kind cluster exists, its
control-plane node is `Ready`, Kubernetes system Pods are running, and
k9s displays resources across all namespaces.
