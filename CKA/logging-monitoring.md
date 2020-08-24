# Logging & Monitoring

**Topics**
-   [Understand how to monitor all cluster components](#understand-how-to-monitor-all-cluster-components)
-   [Understand how to monitor applications](#understand-how-to-monitor-applications)
-   [Manage cluster component logs](#manage-cluster-component-logs)
-   [Manage application logs](#manage-application-logs)
---

## Understand how to monitor all cluster components.
### Resource metrics pipeline
-   provided by `metrics-server` -> https://github.com/kubernetes-sigs/metrics-server.git
-   once installed, it takes a while till metrics are available
-   metrics exposed via `metrics.k8s.io` API
-   metrics are utilized for example by `kubectl top` utility
    -   `kubectl top pod`
    -   `kubectl top node`
-   installation guide:
    ```bash
    git clone https://github.com/kubernetes-sigs/metrics-server.git
    kubectl apply -f metrics-server/deploy/1.8+/
    ```
### Full metrics pipeline
-   provided for example by Prometheus
-   out of scope for CKA (not covered by K8s documentation at all)
-   using either `custom.metrics.k8s.io` or `external.metrics.k8s.io`

### Documentation materials 
-   [Tools for Monitoring Resources](https://v1-16.docs.kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/)
-   [GITHUB: metrics-server](https://github.com/kubernetes-sigs/metrics-server.git)
---

## Understand how to monitor applications.
Kubernetes itself does not provide much sofisticated methods of application monitoring. This layer should be covered by the application code exposing metrics which can be scraped by Prometheus and react on it.

However, as a very simple level of monitoring, we can consider probes.
-   Liveness
    -   monitor the container based on a specific check (http, tcp get, command)
    -   if the check fails, K8s will restart the container
-   Readiness
    -   monitor the container based on a specific check (http, tcp get, command)
    -   if the check fails, K8s will remove the Pod's endpoint from Service -> makes the Pod NotReady

## Manage cluster component logs.
This guide refers to a cluster built via `kubeadm` tool:
-   components running inside a Pod
    -   logs for them can be obtained by:
        -   `kubectl logs -n kube-system <pod_name>`
        -   on respective VM in `/var/log/containers`
    -   kube-scheduler
    -   kube-controller-manager
    -   kube-apiserver
    -   kube-proxy
-   components running as a systemd service
    -   kubelet
        -   logs available via `systemctl -u kubelet` on each VM

There are also some well known locations for logs, but these are not applicable to `kubeadm` cluster:
-   /var/log/kube-apiserver.log
-   /var/log/kube-scheduler.log
-   /var/log/kube-controller-manager.log
-   /var/log/kubelet.log
-   /var/log/kube-proxy.log

Log locations differs per engine used to bootstrap the cluster.
RKE logs:
-   /var/lib/rancher/rke/log/kube-apiserver*.log
-   /var/lib/rancher/rke/log/kube-scheduler*.log
-   /var/lib/rancher/rke/log/kube-controller-manager*.log
-   /var/lib/rancher/rke/log/kubelet*.log
-   /var/lib/rancher/rke/log/kube-proxy*.log

### Documentation materials 
-   [Troubleshoot Clusters](https://v1-16.docs.kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/)
---

## Manage application logs.
From Kubernetes perspective, there is `kubectl logs` utility available for showing current logs.
```bash
# show logs for currently running container in a Pod
kubectl logs <pod_name>

# show logs for previous container in a Pod
kubectl logs <pod_name> --previous

# show continuous log output from current container
kubectl logs -f <pod_name>

# show logs from specific container in a multi-container Pod
kubectl logs <pod_name> -c <container_name>

# show logs from all pods matching the label
kubectl logs -l key=value
```

One can access the logs also on each VM where Pod is cheduled & running. This differs per container runtime used, but for Docker, paths are following:
-   `/var/lib/docker/containers/<container_id>/<container_id>-json.log`
-   `<container_id>` can be easily obtained by calling `docker ps`

### Documentation materials 
-   [Kubectl CheatSheet - Logs](https://v1-16.docs.kubernetes.io/docs/reference/kubectl/cheatsheet/#interacting-with-running-pods)
---