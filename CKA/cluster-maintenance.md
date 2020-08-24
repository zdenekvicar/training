# Cluster maintenance

**Topics**
-   [Understand Kubernetes cluster upgrade process](#understand-kubernetes-cluster-upgrade-process)
-   [Facilitate operating system upgrades](#facilitate-operating-system-upgrades)
-   [Implement backup and restore methodologies](#implement-backup-and-restore-methodologies)
---

## Understand Kubernetes cluster upgrade process.
Since we are running our learning cluster on v1.15.5, we will show how to update it to v1.16.x.
Please note the steps are the same also for v1.14.x -> 1.15.x
Version update limitations
-   Upgrade is allowed only 1 version ahead
    -   v1.14.0 -> v1.15.0
    -   v1.14.0 -> v1.16.0 is `NOT SUPPORTED`

Upgrade steps
-   Determine which version to upgrade to
    ```bash
    apt update && apt-cache policy kubeadm
    # lets pick the recommended candidate version from output of above command -> 1.16.3-00
    kubeadm:
        Installed: 1.15.5-00
        Candidate: 1.16.3-00
    ```
-   Upgrading control plane nodes
    -   `master1`
        ```bash
        # update kubeadm
        apt-mark unhold kubeadm \
         && apt-get update \
         && apt-get install -y kubeadm=1.16.3-00 \
         && apt-mark hold kubeadm
        
        # verify update worked
        kubeadm version

        # run kubeadm upgrade plan
        kubeadm upgrade plan

        # output is as below
            Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
                COMPONENT   CURRENT       AVAILABLE
                Kubelet     6 x v1.15.5   v1.16.3
            Upgrade to the latest stable version:
                COMPONENT            CURRENT   AVAILABLE
                API Server           v1.15.5   v1.16.3
                Controller Manager   v1.15.5   v1.16.3
                Scheduler            v1.15.5   v1.16.3
                Kube Proxy           v1.15.5   v1.16.3
                CoreDNS              1.3.1     1.6.2
                Etcd                 3.3.10    3.3.15-0
            You can now apply the upgrade by executing the following command:
                kubeadm upgrade apply v1.16.3
        
        # now the upgrade part, we have 2 options
        # 1. refresh certificate
            kubeadm upgrade apply v1.16.3
        # 2. keep the cert unchanged <-- lets pick this one as the cluster is new and we do not need to refresh certs
            kubeadm upgrade apply v1.16.3 --certificate-renewal=false
        
        # update CNI plugin
        kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/canal.yaml
        # NOTE: since we are running the same version of plugin already, there is no need to perform this task now 
        ```
    -   `master2` & `master3`
        ```bash
        # update kubeadm
        apt-mark unhold kubeadm \
         && apt-get update \
         && apt-get install -y kubeadm=1.16.3-00 \
         && apt-mark hold kubeadm

        # verify update worked
        kubeadm version

        # run kubeadm upgrade node
        kubeadm upgrade node
        ```
    -   `master1` & `master2` & `master3`
        ```bash
        # upgrade kubelet
        apt-mark unhold kubelet kubectl \
         && apt-get update \
         && apt-get install -y kubelet=1.16.3-00 kubectl=1.16.3-00 \
         && apt-mark kubelet kubectl

        # restart kubelet service
        systemctl restart kubelet
        ```
-   Upgrade worker nodes
    ```bash
    # update kubeadm
    apt-mark unhold kubeadm \
     && apt-get update \
     && apt-get install -y kubeadm=1.16.3-00 \
     && apt-mark hold kubeadm

    # verify update worked
    kubeadm version

    # evict node
    kubectl drain {node} --ignore-daemonsets --delete-local-data --force

    # upgrade kubelet configuration
    kubeadm upgrade node

    # upgrade kubelet
    apt-mark unhold kubelet kubectl \
     && apt-get update \
     && apt-get install -y kubelet=1.16.3-00 kubectl=1.16.3-00 \
     && apt-mark kubelet kubectl

    # restart kubelet service
    systemctl restart kubelet

    # uncordon nodes
    kubectl uncordon {node}
    ```
-   Verify the status of the cluster
    ```bash
    kubectl get nodes -o wide
    ```
-   Recovering from a failure state
    ```bash
    # in case of failure of `kubectl upgrade`, command can be just re-run
    kubectl upgrade node # for 2nd+ master and workers
    kubeadm upgrade apply v1.16.3 # for 1st master, with cert renew
    kubeadm upgrade apply v1.16.3 --certificate-renewal=false # for 1st master, without cert renew

    # in case a cluster is in a bad / error shape, it can be narrowed down by
    kubeadm upgrade apply --force
    # this will re-apply current version and should fix the cluster
    ```
-   How it works
    -   `kubeadm upgrade apply`
        -   Checks that your cluster is in an upgradeable state:
            -   The API server is reachable
            -   All nodes are in the Ready state
            -   The control plane is healthy
        -   Enforces the version skew policies.
        -   Makes sure the control plane images are available or available to pull to the machine.
        -   Upgrades the control plane components or rollbacks if any of them fails to come up.
        -   Applies the new `kube-dns` and `kube-proxy` manifests and makes sure that all necessary RBAC rules are created.
        -   Creates new certificate and key files of the API server and backs up old files if they’re about to expire in 180 days.
    -   `kubeadm upgrade node` on additional control plane nodes
        -   Fetches the kubeadm `ClusterConfiguration` from the cluster.
        -   Optionally backups the `kube-apiserver` certificate.
        -   Upgrades the static Pod manifests for the control plane components.
        -   Upgrades the kubelet configuration for this node.
    -   `kubeadm upgrade node` on worker nodes
        -   Fetches the kubeadm ClusterConfiguration from the cluster.
        -   Upgrades the kubelet configuration for this node.

### Documentation materials:
-   [Upgrading kubeadm clusters](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
---

## Facilitate operating system upgrades.
Before any OS upgrade, K8s node should be 'released' from the cluster to not affect any running workloads. Therefore the process would be as follows:
1.  Evict node
    ```bash
    # evict node
    # This will make the node Unschedulable + drains running Pods
    kubectl drain {node} --ignore-daemonsets --delete-local-data --force

    # just set Unschedulable without draining any running pods
    kubectl cordon node
    ```
2.  Perform OS upgrade
3.  Uncordon node
    ```bash
    # make the node schedulable again
    kubectl uncordon {node}
    ```
---

## Implement backup and restore methodologies.
Below is a working guide how to backup & destroy & restore single master K8s cluster built with kubeadm
1.  Backup PKI foler
    ```bash
    # backup whole folder /etc/kubernetes/pki
    mkdir /tmp/cluster_backup
    cp -r /etc/kubernetes/pki/ /tmp/cluster_backup/
    ```
2.  Backup ETCD

    `NOTE: DO NOT FORGET to specify ETCDCTL_API=3 variable before working with etcd cluster`
    ```bash
    # on any master node running etcd:
    cat /etc/kubernetes/manifests/etcd.yaml | egrep -w 'advertise-client-urls|peer-trusted-ca-file|peer-cert-file|peer-key-file'

    # expected output will be like this
        - --advertise-client-urls=https://172.20.125.46:2379
        - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
        - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
        - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt

    # now exec into etcd pod
    kubectl exec -it -n kube-system etcd-master1 sh

    # take snapshot of ETCD DB
    # use details from commands above
    ETCDCTL_API=3 etcdctl snapshot save /var/lib/etcd/snapshot.db \
    --endpoints=https://172.20.125.46:2379\
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/peer.crt \
    --key=/etc/kubernetes/pki/etcd/peer.key

    # verify snapshot details
    ETCDCTL_API=3 etcdctl --write-out=table snapshot status /var/lib/etcd/snapshot.db
    +----------+----------+------------+------------+
    |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
    +----------+----------+------------+------------+
    | 84d89a18 |     1270 |       1278 |     1.6 MB |
    +----------+----------+------------+------------+
    ```
3.  Destroy master node
    ```bash
    kubeadm reset
    rm -rf /var/lib/etcd
    ```
4.  Restore PKI folder
    ```bash
    mkdir /etc/kubernetes
    cp -r /tmp/cluster_backup/pki/ /etc/kubernetes/
    ```
5.  Restore ETCD
    ```bash
    # prepare etcd folder
    mkdir -p /var/lib/etcd

    # run docker with etcd container, starting the restoration process
    docker run --rm \
    -v '/tmp/cluster_backup:/backup' \
    -v '/var/lib/etcd:/var/lib/etcd' \
    --env ETCDCTL_API=3 \
    'k8s.gcr.io/etcd-amd64:3.1.12' \
    /bin/sh -c "etcdctl snapshot restore '/backup/snapshot.db' ; mv /default.etcd/member/ /var/lib/etcd/"
    ```
6.  Re-initialize cluster
    ```bash
    # ignores non empty /var/lib/etcd which -> we already filled it with data
    # uses present /etc/kubernetes/pki certificates
    kubeadm init --ignore-preflight-errors=DirAvailable--var-lib-etcd

    # Once the command finishes, cluster is available again, all cluster nodes will automatically reconnect to this master (certs are the same, address also)
    ```
### Documentation materials:
-   [Operating etcd clusters for Kubernetes](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
-   Not usable during CKA (not in `kubernetes.io` domain)
    -   [Backup and Restore a Kubernetes Master with Kubeadm](https://labs.consol.de/kubernetes/2018/05/25/kubeadm-backup.html)
---