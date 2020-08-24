# Troubleshooting

**Topics**
-   [Troubleshoot application failure](#troubleshoot-application-failure)
-   [Troubleshoot control plane failure](#troubleshoot-control-plane-failure)
-   [Troubleshoot worker node failure](#troubleshoot-worker-node-failure)
-   [Troubleshoot networking](#troubleshoot-networking)
---

## Troubleshoot application failure
-   Debug Pods
    -   Pending
        -   Pod cannot be scheduled into a Node
        -   utilize `kubectl describe` to see latest events
        -   most common cases:
            -   insufficient resources
                -   add more resources
                -   adjust resource quotas
                -   remove old pods
            -   hostPort
                -   think of using Service instead
                -   number of Pods with hostPort cannot be higher than number of Nodes in cluster
    -   Waiting
        -   Pod was scheduled to a Node, but cannot run here
        -   utilize `kubectl describe` to see latest events
        -   3 initial things to check:
            -   docker image name is correct
            -   docker image is pushed to the target repository
            -   docker image can be manually pulled on your machine (or on worker node if you have access there)
            -   4th, and most important -> `read events from kubectl describe` ;-)
    -   Crash or unhealthy
        -   check container logs `kubectl logs <pod> <container>`
        -   check logs of previous container `kubectl logs <pod> <container> --previous`
        -   exec into container to investigate more `kubectl exec -it <pod> <container> -- <command> <arg1> ... <argn>`
-   Debug Replication controllers
    -   not much logic here, if RC does not work -> investigate the `Pod` section above
    -   `kubectl describe rc <name>` can be used as well to see latest events
-   Debug Services
    -   verify endpoints
        -   `kubectl get endpoints <service_name>`
        -   number of endpoints should match with number of Pods covered by service
        -   IPs of endpoints should match with IPs of Pods covered by service
    -   missing endpoints
        -   service is matching pods using labels
        -   try to get all pods wich refered labels as the service would do
            -   `kubectl get pods --selector=<key>=<value>`
        -   verify service is referring to same ports as exposed on Pods
    -   network traffic is not forwarded
        -   if you can connect, but the connection is dropped instantly, could be because of proxy cannot contact Pod
            -   verify Pod health
            -   try to reach Pod directly using its IP
            -   verify port on service matches the port exposed in Pod
    -   verify service DNS
        -   try to access the service from Pod by its hostname `nslookup <service_name>`
        -   if does not work, try to access it with namespace `nslookup <service_name>.<namespace>`
        -   try FQDN `nslookup <service_name>.<namespace>.svc.cluster.local`
        -   if FQDN only works, then check your `/etc/resolv.conf` inside the Pod
            -   nameserver detail has to match with DNS service IP
                -   DNS service IP is passed to `kubelet` with `--cluster-dns` flag
            -   search must indicate all layers of searching
                -   check all services in local namespace `<namespace>.svc.cluster.local`
                -   check all services in all namespaces `svc.cluster.local`
                -   check in whole cluster `cluster.local`
            -   cluster suffix can be found from `kubelet` args under flag `--cluster-domain`
        -   if all above fails, check accesibility of kubernetes master service, which should be accessible all the time in case DNS lookup it not in fatal state
            -   `nslookup kubernetes.default`
            -   if this fails, investigate `kube-proxy` and DNS itself
-   Debug kube-proxy
    -   check if kube-proxy is running on all nodes
        -   `ps auxw | grep kube-proxy`
    -   check logs of `kube-proxy` (journalctl or /var/log/kube-proxy.log ... depends on OS and k8s engine used to build a cluster)
        -   look for any errors about contacting K8s master
    -   check if kube-proxy can write to:
        -   iptables
            -   `iptables-save | grep KUBE`
        -   IPVS
            -   `ipvsadm -ln`
            -   here you should see IP of each service + all child Pod IPs

### Documentation materials:
-   [Troubleshoot Applications](https://v1-16.docs.kubernetes.io/docs/tasks/debug-application-cluster/debug-application/)
-   [Debug Services](https://v1-16.docs.kubernetes.io/docs/tasks/debug-application-cluster/debug-service/)
---

## Troubleshoot control plane failure
-   Check nodes status
    -   `kubectl get nodes -o wide`
    -   `kubectl describe node <node>`
-   Check control-plane components
    -   check services
        -   service-based components
            ```bash
            $ service kube-apiserver status
            $ service kube-controller-manager status
            $ service kube-scheduler status
            ```
        -   some k8s clusters are running services as a containers on VMs, in that case, check the containers health
    -   check logs
        -   `journalctl` if available
        -   api-server: `/var/log/kube-apiserver.log`
        -   kube-scheduler: `/var/log/kube-scheduler.log`
        -   kube-controller-manager: `/var/log/kube-controller-manager.log`
        -   log location is based on k8s engine used to build a cluster

## Troubleshoot worker node failure
-   Check node status
    -   `kubectl get nodes -o wide`
    -   `kubectl describe node <node>`
-   Check worker components
    -   check services
        -   service-based components
            ```bash
            $ service kubelet status
            $ service kube-proxy status
            ```
        -   some k8s clusters are running services as a containers on VMs, in that case, check the containers health
    -   check logs
        -   `journalctl` if available
        -   kubelet: `/var/log/kubelet.log`
        -   kube-proxy: `//var/log/kube-proxy.log`
        -   log location is based on k8s engine used to build a cluster

## Troubleshoot networking
-   Make sure you’re connecting to the service’s cluster IP from within the cluster, not from the outside.
-   Don’t bother pinging the service IP to figure out if the service is accessible (remember, the service’s cluster IP is a virtual IP and pinging it will never work).
-   If you’ve defined a readiness probe, make sure it’s succeeding; otherwise the pod won’t be part of the service.
-   To confirm that a pod is part of the service, examine the corresponding Endpoints object with `kubectl get endpoints`.
-   If you’re trying to access the service through its FQDN or a part of it (for example, `myservice.mynamespace.svc.cluster.local` or `myservice.mynamespace`) and it doesn’t work, see if you can access it using its cluster IP instead of the FQDN.
-   Check whether you’re connecting to the port exposed by the service and not the target port.
-   Try connecting to the pod IP directly to confirm your pod is accepting connections on the correct port.
-   If you can’t even access your app through the pod’s IP, make sure your app isn’t only binding to localhost.