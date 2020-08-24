# Installation & Configuration & Validation

**Topics**
-   [Design a Kubernetes cluster](#design-a-kubernetes-cluster)
-   [Install Kubernetes masters and nodes](#install-kubernetes-masters-and-nodes)
-   [Configure secure cluster communications](#configure-secure-cluster-communications)
-   [Configure a Highly-Available Kubernetes cluster](#configure-a-highly-available-kubernetes-cluster)
-   [Know where to get the Kubernetes release binaries](#know-where-to-get-the-kubernetes-release-binaries)
-   [Provision underlying infrastructure to deploy a Kubernetes cluster](#provision-underlying-infrastructure-to-deploy-a-kubernetes-cluster)
-   [Choose a network solution](#choose-a-network-solution)
-   [Choose your Kubernetes infrastructure configuration](#choose-your-kubernetes-infrastructure-configuration)
-   [Run end-to-end tests on your cluster](#run-end-to-end-tests-on-your-cluster)
-   [Analyse end-to-end tests results](#analyse-end-to-end-tests-results)
-   [Run Node end-to-end tests](#run-node-end-to-end-tests)
-   [Install and use kubeadm to install, configure, and manage Kubernetes clusters](#install-and-use-kubeadm-to-install-configure-and-manage-kubernetes-clusters)
---

## Design a Kubernetes cluster.
-   Dev cluster setup
    -   possibly utilize `minikube`
    -   1x master
    -   1-n workers
    -   `control plane` with `worker plane` can run on single node
-   HA setup
    -   3x masters
    -   3-n workers
-   PROD HA setup
    -   5x masters
    -   3-n workers
    -   2 system workers (ingress + cluster-wide resources like monitoring, ...)

---
## Install Kubernetes masters and nodes.
This guide is wrote for `kubeadm` usage. This section covers bootstraping of single control plane cluster (Dev setup).
-   **`Create 2 VMs taking into considerations below specs`**:
    -   Supported OS
        -   Ubuntu 16.04+
        -   CentOS / RHEL / OL 7+
            -   RHEL 8 is not compatible with latest `kubeadm`
        -   Debian 9
    -   Sizing
        -   RAM: 2+ GB
        -   CPU: 2+
    -   Other requirements
        -   full network connectivity between both VMs
        -   both VMs has to have unique MAC address and product_uuid
            -   MAC: `ifconfig -a`
            -   product_uuid: `sudo cat /sys/class/dmi/id/product_uuid`
        -   K8s used ports are opened
            -   Control plane:
                -   `6443`: Kubernetes API server
                -   `2379,2380`: etcd
                -   `10250`: Kubelet API
                -   `10251`: kube-scheduler
                -   `10252`: kube-controller-manager
            -   Worker nodes:
                -   `10250`: Kubelet API
                -   `30000-23767`: NodePort services
        -   SWAP disabled
        -   iptables running in `legacy` mode
            -   required to check on:
                -   Debian 10+
                -   Ubuntu 19.04 +
                -   Fedora 29+
            ``` bash
            # Debian / Ubuntu
            update-alternatives --set iptables /usr/sbin/iptables-legacy
            update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
            update-alternatives --set arptables /usr/sbin/arptables-legacy
            update-alternatives --set ebtables /usr/sbin/ebtables-legacy
            ```
-  **`Install container runtime`**
    -   For Kubernetes 1.15, Docker version 1.13+ has to be used
    ```bash
    # Install Docker CE
    ## Set up the repository:
    ### Install packages to allow apt to use a repository over HTTPS
    apt-get update && apt-get install -y apt-transport-https ca-certificates curl software-properties-common

    ### Add Docker’s official GPG key
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

    ### Add Docker apt repository.
    add-apt-repository \
        "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
        $(lsb_release -cs) \
        stable"

    ## Install Docker CE -> this will install latest stable version
    apt-get update && apt-get install -y docker-ce

    ## Check available versions if needed
    apt-cache madison docker-ce

    # Setup daemon.
    cat > /etc/docker/daemon.json <<EOF
    {
        "exec-opts": ["native.cgroupdriver=systemd"],
        "log-driver": "json-file",
        "log-opts": {
            "max-size": "100m"
        },
        "storage-driver": "overlay2"
    }
    EOF

    mkdir -p /etc/systemd/system/docker.service.d

    # Restart docker.
    systemctl daemon-reload
    systemctl restart docker

    # Verify installation 
    docker run hello-world
    ```
-   **`Installing kubeadm, kubelet, kubectl`**
    ```bash
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -

    cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
        deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF

    apt-get update && apt-get install -y kubelet=1.15.5-00 kubeadm=1.15.5-00 kubectl=1.15.5-00

    apt-mark hold kubelet kubeadm kubectl
    ```
-   **`Initializing control-plane`**
    ```bash
    # Pre-pull images
    kubeadm config images pull

    # init
    kubeadm init --pod-network-cidr=10.244.0.0/16

    # sometimes you are forced to use --config option and use external configuration file
    # if that is the case -> here is the syntax: https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta2#ClusterConfiguration
    # example:

    apiVersion: kubeadm.k8s.io/v1alpha3
    kind: ClusterConfiguration
    networking:
      podSubnet: 10.244.0.0/16

    # verification
    cp /etc/kubernetes/admin.conf ~/.kube/config
    kubectl get node
    ```
-   **`Installing network plugin`**
    ```bash
    # install Calico
    kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/canal.yaml

    # verification
    kubectl get pods --all-namespaces
    ```
-   **`Joining worker node`**
    ```bash
    # Join worker node (run as root on Worker VM)
    kubeadm join \
    --token <token> <master-ip>:<master-port>\
    --discovery-token-ca-cert-hash sha256:<hash>

    # Get current kubeadm token
    kubeadm token list

    # Get discovery-token-ca-cert-hash
    openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
    openssl dgst -sha256 -hex | sed 's/^.* //'

    # Verification
    kubectl get nodes
    ```
-   **`Teardown`**
    ```bash
    # first we need to drain a node we are going to remove
    kubectl drain <node name> --delete-local-data --force --ignore-daemonsets

    # then we can delete it from cluster
    kubectl delete node <node name>

    # reset all kubeadm settings on removed node
    kubeadm reset

    # reset networking stuff on VM (iptables and IPVS tables)
    iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
    ipvsadm -C
    ```

### Documentation materials:
-   [Installing kubeadm](https://v1-16.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
-   [Creating single control plane cluster](https://v1-16.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
-   [kubeadm init](https://v1-16.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-init/)
-   [kubeadm join](https://v1-16.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/)
-   [kubeadm reset](https://v1-16.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-reset/)
-   [kubeadm token](https://v1-16.docs.kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)
---
## Configure secure cluster communications.
Using `kubeadm` will automatically setup TLS configuration between Kubernetes components for us, so there is no need to bootstrap it manually. However should such case occurs, use this guide from K8s documentation.

### Documentation materials
-   [TLS bootstraping](https://v1-16.docs.kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)

---
## Configure a Highly-Available Kubernetes cluster.
We have `two options` when deciding how to setup HA cluster:
1. Stacked control-plane and etcd nodes
2. External etcd nodes

In this guide, we are running `stacked version` to minimize the need for VMs. We will be usind the same master nodes to run control-plane components as well as etcd instances

Below are steps to build HA option for K8s cluster.
-   **`Configure kube-apiserver LoadBalancer`**
    ```bash
    # Login to the LoadBalancer node and prepare HAproxy packages
    sudo apt-get update ; sudo apt-get install -y apache2 haproxy
    
    # Adjust configuration file of HAproxy
    cat > /etc/haproxy/haproxy.cfg <<EOF
    global
        log /dev/log	local0
        log /dev/log	local1 notice
        chroot /var/lib/haproxy
        stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
        stats timeout 30s
        user haproxy
        group haproxy
        daemon

        # Default SSL material locations
        ca-base /etc/ssl/certs
        crt-base /etc/ssl/private

        # Default ciphers to use on SSL-enabled listening sockets.
        # For more information, see ciphers(1SSL). This list is from:
        #  https://hynek.me/articles/hardening-your-web-servers-ssl-ciphers/
        # An alternative list with additional directives can be obtained from
        #  https://mozilla.github.io/server-side-tls/ssl-config-generator/?server=haproxy
        ssl-default-bind-ciphers ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:RSA+AESGCM:RSA+AES:!aNULL:!MD5:!DSS
        ssl-default-bind-options no-sslv3

    defaults
        log	 global
        mode tcp
        option tcplog
        option dontlognull
            timeout connect 5000
            timeout client  50000
            timeout server  50000
        errorfile 400 /etc/haproxy/errors/400.http
        errorfile 403 /etc/haproxy/errors/403.http
        errorfile 408 /etc/haproxy/errors/408.http
        errorfile 500 /etc/haproxy/errors/500.http
        errorfile 502 /etc/haproxy/errors/502.http
        errorfile 503 /etc/haproxy/errors/503.http
        errorfile 504 /etc/haproxy/errors/504.http
    
    frontend k8s
        bind *:6443
        default_backend k8s_masters
    
    backend k8s_masters
        balance roundrobin
        server master1 172.20.125.46:6443 check
    EOF

    # reload HAproxy 
    systemctl restart haproxy.service

    # verify it runs
    systemctl status haproxy.service
    ```
-   **`Update first control plane node`**
    -   On the first control plane node, create a configuration file called `kubeadm-config.yml`
        ```yaml
        apiVersion: kubeadm.k8s.io/v1beta2
        kind: ClusterConfiguration
        kubernetesVersion: v1.15.5
        controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT"
        networking:
          podSubnet: 192.168.0.0/16
        ```
    -   Initialize first node
        ```bash
        kubeadm init --config=kubeadm-config.yml --upload-certs
        ```
    -   Install network plugin
        ```bash
        kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/canal.yaml
        ```
    -   Verification
        ```bash
        cp /etc/kubernetes/admin.conf ~/.kube/config
        kubectl get node
        kubectl get pods --all-namespaces
        ```

-   **`Join other control plane nodes`**
    -   Jonining other master nodes should be very easy now, as previous `kubeadm init` should return exact command to do so, however below are steps just for sure
    ```bash
    # Join another master node
    kubeadm join \
    --token <token> <loadbalancer-ip/dns>:<loadbalancer-port>\
    --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <cert_key>

    # certificates will be uploaded to every new master node added, however there is an option how to re-generate them
    kubeadm init phase upload-certs --upload-certs
    ```
-   **`Add worker nodes`**
    -   Join worker nodes with commands received from `kubeadm init` on first master node
        ```bash
        # Join worker node (run as root on Worker VM)
        kubeadm join \
        --token <token> <master-ip>:<master-port>\
        --discovery-token-ca-cert-hash sha256:<hash>

        # Get current kubeadm token
        kubeadm token list

        # Get discovery-token-ca-cert-hash
        openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
        openssl dgst -sha256 -hex | sed 's/^.* //'

        # Verification
        kubectl get nodes
        ```

### Documentation materials:
-   [Creating Highly Available clusters with kubeadm
](https://v1-16.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
---
## Know where to get the Kubernetes release binaries.
Binaries for every K8s verison can be `found on release notes page`, in `Downloads` section.
-   [v1.16](https://v1-16.docs.kubernetes.io/docs/setup/release/notes/#downloads-for-v1-16-0)

---
## Provision underlying infrastructure to deploy a Kubernetes cluster.
This task highly depends on infrastructure provided one is using. Just keep in mind some minimum specifications (increased based on your needs, i.e. total number of cluster nodes, etc.)

### Documentation materials:
-   [Size of master and master components](https://v1-16.docs.kubernetes.io/docs/setup/best-practices/cluster-large/#size-of-master-and-master-components)
-   [VM requirements](https://v1-16.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#before-you-begin)

---
## Choose a network solution.
In this setup, we are going to use Canal. Canal uses:
-   Calico for policies
-   Flannel for networking

It does require a specific argument to be sent to `kubeadm init` in order to work:
-   `--pod-network-cidr=10.244.0.0/16`

It also works only on `amd64` machines only.

---
## Choose your Kubernetes infrastructure configuration.
Basically the same topic as covered 2 sections above in [Provision underlying infrastructure to deploy a Kubernetes cluster](#provision-underlying-infrastructure-to-deploy-a-kubernetes-cluster)

---
## Run end-to-end tests on your cluster. 
Verify that you can run these checked items:

1. Deployments + Pods can run
    ```bash
    # run from any node with kubeconfig configured
    kubectl run nginx --image=nginx
    kubectl get deployments
    kubectl get pods --all-namespaces
    ```
2. Pods can be directly accessed
    ```bash
    # check where nginx pod from 1. is running
    # configure .kube/config on that worker node (copy from master)
    # create a port forwarding on that worker node
    kubectl port-dorward <nginx_podname> 8081:80

    # create new session on same worker node and test connection
    curl --head http://127.0.0.1:8081
    ```
3. Logs can be collected
    ```bash
    kubectl logs <nginx_podname>
    ```
4. Commands run from Pod
    ```bash
    kubectl exec -it <nginx_podname> env
    ```
5. Services can provide access
    ```bash
    # expose the deployment via NodePort service
    kubectl expose deployment nginx --port 80 --type NodePort

    # confirm it works (from node where pod is running)
    # find out node port created
    kubectl get service --all-namespaces
    curl -T localhost:<node_port>
    ```
6. Nodes are healthy
    ```bash
    kubectl get nodes -o wide
    kubectl describe nodes | grep -A 6 "Conditions:"
    kubectl describe nodes | grep -A 10 "Events:"
    ```
7. Pods are healthy
    ```bash
    kubectl get pods --all-namespaces
    kubectl describe pods
    ```
---
## Analyse end-to-end tests results.
Some sources recognizes this section as an testing of Kubernetes itself, and pointing to [kubetest](k8s.io/test-infra/kubetest) project.

I get it as an general understanding of results obtained from section above ... getting all the info about the cluster and its health.

---
## Run Node end-to-end tests.
Several stels intended to ensure Node components are running fine:
-   using `kubectl`
    ```bash
    kubectl get pods
    kubectl get pods -n kube-system
    kubectl run nginx --image=nginx
    kubectl scale replicas=3 deploy/nginx
    ```
-   checking directly on node VM
    -   usable only in case components are running as a services on VM
    -   if components are running as a containers, they can be checked using container runtime CLI tool
    ```bash
    # from control-plane node
    service kube-apiserver status
    service kube-controller-manager status
    service kube-scheduler status
    # from worker node
    service kubelet status
    service kube-proxy status
    ```

---
## Install and use kubeadm to install, configure, and manage Kubernetes clusters
This topic was described several time in this document already.
-   [Install Kubernetes masters and nodes](#install-kubernetes-masters-and-nodes)
-   [Configure a Highly-Available Kubernetes cluster](#configure-a-highly-available-kubernetes-cluster)