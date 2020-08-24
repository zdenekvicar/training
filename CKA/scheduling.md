# Scheduling

**Topics**
-   [Use label selectors to schedule Pods](#use-label-selectors-to-schedule-pods)
-   [Understand the role of DaemonSets](#understand-the-role-of-daemonsets)
-   [Understand how resource limits can affect Pod scheduling](#understand-how-resource-limits-can-affect-pod-scheduling)
-   [Understand how to run multiple schedulers and how to configure Pods to use them](#understand-how-to-run-multiple-schedulers-and-how-to-configure-pods-to-use-them)
-   [Manually schedule a pod without a scheduler](#manually-schedule-a-pod-without-a-scheduler)
-   [Display scheduler events](#display-scheduler-events)
-   [Know how to configure the Kubernetes scheduler](#know-how-to-configure-the-kubernetes-scheduler)
---

## Use label selectors to schedule Pods.
```bash
# label a node
kubectl label node worker2 name=worker2

# prepare deployment manifests
kubectl create nginx-worker2 --image=nginx --replicas=5 --dry-run -o yaml > deploy_worker2.yml

# edit manifest to apply label selectors
vim deploy_worker2.yml

# specified in .spec.template.spec.nodeSelector
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    run: nginx-worker2
  name: nginx-worker2
spec:
  replicas: 5
  selector:
    matchLabels:
      run: nginx-worker2
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: nginx-worker2
    spec:
      containers:
      - image: nginx
        name: nginx-worker2
        resources: {}
      nodeSelector: 
        name: worker2
status: {}

# apply manifests and verify where are they scheduled
kubectl apply -f deploy_worker2.yml
kubectl get pods -o wide -w
```

---
## Understand the role of DaemonSets.
### Introduction
- A DaemonSet ensures that all (or some) Nodes run a copy of a Pod. 
- As nodes are added to the cluster, Pods are added to them. 
- As nodes are removed from the cluster, those Pods are garbage collected. 
- Deleting a DaemonSet will clean up the Pods it created.

### Manifest
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: myapp-ds
spec:
  selector:
    matchLabels: 
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx-container
          image: nginx
```

### Scheduling of DaemonSets
- Scheduled by DaemonSet controller 
  - disabled by default since 1.12
  - used when `.spec.nodeName` is populated, therefore ignored by Scheduler
  - the unschedulable field of a node is not respected by the DaemonSet controller
  - the DaemonSet controller can make Pods even when the scheduler has not been started, which can help cluster bootstrap
- Scheduled by default scheduler
  - enabled by default since 1.12
  - used when `NodeAffinity` is used instead of `.spec.nodeName`
  - the default scheduler is then used to bind the pod to the target host


### Documentation materials 
- [DaemonSet](https://v1-16.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/)

---
## Understand how resource limits can affect Pod scheduling.
- resource limits can be enforced by object `ResourceQuota`
- applied to specific namespace
- manifest:
  ```yaml
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: mem-cpu-demo
  spec:
    hard:
      requests.cpu: "1"
      requests.memory: 1Gi
      limits.cpu: "2"
      limits.memory: 2Gi
  ```
  - above manifests will enforce the following:
    - Every Container must have a memory request, memory limit, cpu request, and cpu limit.
    - The memory request total for all Containers must not exceed 1 GiB.
    - The memory limit total for all Containers must not exceed 2 GiB.
    - The CPU request total for all Containers must not exceed 1 cpu.
    - The CPU limit total for all Containers must not exceed 2 cpu.
- Practical part:
  ```bash
  # create namespace
  kubectl create namespace quota-mem-cpu-example

  # apply resourceQuota
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: mem-cpu-demo
    namespace: quota-mem-cpu-example
  spec:
    hard:
      requests.cpu: "1"
      requests.memory: 1Gi
      limits.cpu: "2"
      limits.memory: 2Gi
  EOF

  # Check resourceQuota
  kubectl describe resourcequota -n quota-mem-cpu-example
  Name:            mem-cpu-demo
  Namespace:       quota-mem-cpu-example
  Resource         Used  Hard
  --------         ----  ----
  limits.cpu       0     2
  limits.memory    0     2Gi
  requests.cpu     0     1
  requests.memory  0     1Gi

  # Apply 1st pod
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Pod
  metadata:
    name: quota-mem-cpu-demo
    namespace: quota-mem-cpu-example
  spec:
    containers:
    - name: quota-mem-cpu-demo-ctr
      image: nginx
      resources:
        limits:
          memory: "800Mi"
          cpu: "800m" 
        requests:
          memory: "600Mi"
          cpu: "400m"
  EOF

  # Check resourceQuota
  kubectl describe resourcequota -n quota-mem-cpu-example
  Name:            mem-cpu-demo
  Namespace:       quota-mem-cpu-example
  Resource         Used   Hard
  --------         ----   ----
  limits.cpu       800m   2
  limits.memory    800Mi  2Gi
  requests.cpu     400m   1
  requests.memory  600Mi  1Gi

  # Apply 2nd pod
  cat <<EOF | kubectl apply -f -
  apiVersion: v1
  kind: Pod
  metadata:
    name: quota-mem-cpu-demo-2
    namespace: quota-mem-cpu-example
  spec:
    containers:
    - name: quota-mem-cpu-demo-2-ctr
      image: redis
      resources:
        limits:
          memory: "1Gi"
          cpu: "800m"      
        requests:
          memory: "700Mi"
          cpu: "400m"
  EOF

  Error from server (Forbidden): error when creating "STDIN": pods "quota-mem-cpu-demo-2" is forbidden: exceeded quota: mem-cpu-demo, requested: requests.memory=700Mi, used: requests.memory=600Mi, limited: requests.memory=1Gi
  ```
### Documentation materials 
- [Configure Memory and CPU Quotas for a Namespace](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/)
---

## Understand how to run multiple schedulers and how to configure Pods to use them.
### Running custom scheduler
Below are just basic steps, as building and running own custom scheduler is considered too much for the CKA (my personal opinion -> Z.Vicar)
- Prepare docker image
- Prepare manifests
  - ServiceAccount
  - ClusterRoleBinding (ref to system:kube-scheduler role)
  - Deployment
- Run the custom scheduler in cluster
  - apply manifests
  - udpate ClusterRole
    - `kubectl edit clusterrole system:kube-scheduler`
    - add `my-scheduler` under `.rules.apiGroups.resourceNames`

### Pods configuration
- specified under `.spec.schedulerName`
- verified by checking Pod events, look for `Scheduled` entries
- manifest configured to use `my-scheduler` 
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: annotation-second-scheduler
    labels:
      name: multischeduler-example
  spec:
    schedulerName: my-scheduler
    containers:
    - name: pod-with-second-annotation-container
      image: k8s.gcr.io/pause:2.0
  ```

### Documentation materials 
- [Configure Multiple Schedulers](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/configure-multiple-schedulers/)
---

## Manually schedule a pod without a scheduler.
- by default, all static pods are stored in `/etc/kubernetes/manifests`
- if one stores a manifest file there, it will be taken by kubelet and started as a staticPod
- example: create a static pod with busybox and sleep command
  - `kubectl run --restart=Never --image=busybox static-busybox --dry-run -o yaml --command -- sleep 1000 > /etc/kubernetes/manifests/static-busybox.yml`
- If you are asked to delete a static Pod from a specific node then:
  ```bash
  # find a IP of a node
  kubectl get nodes -o wide
  
  # ssh to the node and find kubelet config
  systemctl status kubelet # look for --config=/var/lib/kubelet/config.yaml

  # find location of staticPods:
  grep static /var/lib/kubelet/config.yaml
    staticPodPath: /etc/kubernetes/manifests

  # navigate to that directory and remove manfiest file of a Pod required to be removed
  ```

## Display scheduler events.
- Scheduler running as a Pod
  ```bash
  kubectl get events
  kubectl get events -w
  kubectl -n kube-system logs -f kube-scheduler-master1
  ```
- Scheduler running as a service on OS level
  ```bash
  # find details about service
  systemctl status kube-scheduler

  # check if its logging via journalctl
  journalctl -u kube-scheduler

  # possible log file location
  /var/log/kube-scheduler.log
  ```
---

## Know how to configure the Kubernetes scheduler.
This guide is considering kubeadm built cluster on version 1.15.5, which runs kube-scheduler as a staticPod inside a cluster.

- startup flags
  ```bash
  # location of staticPod manifest
  /etc/kubernetes/manifests/kube-scheduler.yaml 

  # inside a manifest, look for .spec.containers.command
  spec:
    containers:
    - command:
      - kube-scheduler
      - --bind-address=127.0.0.1
      - --kubeconfig=/etc/kubernetes/scheduler.conf
      - --leader-elect=true
  ```
### Documentation materials 
- [kube-scheduler](https://v1-16.docs.kubernetes.io/docs/reference/command-line-tools-reference/kube-scheduler/)
