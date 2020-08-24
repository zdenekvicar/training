# Networking

**Topics**
-   [Understand the networking configuration on the cluster nodes](#understand-the-networking-configuration-on-the-cluster-nodes)
-   [Understand Pod networking concepts](#understand-pod-networking-concepts)
-   [Understand service networking](#understand-service-networking)
-   [Deploy and configure network load balancer](#deploy-and-configure-network-load-balancer)
-   [Know how to use Ingress rules](#know-how-to-use-ingress-rules)
-   [Know how to configure and use the cluster DNS](#know-how-to-configure-and-use-the-cluster-dns)
-   [Understand CNI](#understand-cni)
---

## Understand the networking configuration on the cluster nodes.
Whole networking concept is divided into 4 sub-sections:
1.  `Container <-> Container`
    -   `IP-per-Pod` model
        -   containers inside a Pod shares the same IP address
        -   communicates with each other via `localhost`
        -   differentiates between each other with `ports`
        -   implementation of this is very specific to container-runtime
2.  `Pod <-> Pod`
    -   Every `Pod` has its own IP address, because of that:
        -   we dont need to create specific links between `Pods`
        -   `Pods` are much like VMs or physical hosts from the perspective of networking:
            -   port allocation,
            -   naming,
            -   service discovery,
            -   load balancing,
            -   application configuration,
            -   migration
    -   In order to achieve this, Kubernetes expects the following to be true:
        -   `pods on a node` can communicate with `all pods on all nodes` without NAT
        -   `agents on a node` (e.g. system daemons, kubelet) can communicate with `all pods on that node`
        -   `pods in the host network of a node` can communicate with `all pods on all nodes` without NAT
    -   This model can be implemented using different Network plugins. Not all of them are covered in this list. To see more, visit [K8s documentation](https://v1-16.docs.kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)
        -   `Cisco ACI`
            -   integrated overlay and underlay SDN
            -   supports containers, virtual machines, and bare metal servers
        -   `Flannel`
            -   very simple overlay network
        -   `Project Calico`
            -   open source container networking provider and network policy engine
        -   `Canal`
            -   combination of two:
                -   `Calico` for networking
                -   `Flannel` for policy enforcement
        -   `Weave Net from Weaveworks`
            -   runs as a CNI plug-in or stand-alone
            -   doesn’t require any configuration or extra code to run
            -   the network provides one IP address per pod - as is standard for Kubernetes
3.  `Pod <-> Service`
    -   utilizes `Services`
4.  `External <-> Service`
    -   utilizes `Services`
    
### Documentation materials:
-   [Cluster Networking](https://v1-16.docs.kubernetes.io/docs/concepts/cluster-administration/networking/)
---

## Understand Pod networking concepts.
1.  `Container <-> Container`
    -   `IP-per-Pod` model
        -   containers inside a Pod shares the same IP address
        -   communicates with each other via `localhost`
        -   differentiates between each other with `ports`
        -   implementation of this is very specific to container-runtime
2.  `Pod <-> Pod`
    -   Every `Pod` has its own IP address, because of that:
        -   we dont need to create specific links between `Pods`
        -   `Pods` are much like VMs or physical hosts from the perspective of networking:
            -   port allocation,
            -   naming,
            -   service discovery,
            -   load balancing,
            -   application configuration,
            -   migration
    -   In order to achieve this, Kubernetes expects the following to be true:
        -   `pods on a node` can communicate with `all pods on all nodes` without NAT
        -   `agents on a node` (e.g. system daemons, kubelet) can communicate with `all pods on that node`
        -   `pods in the host network of a node` can communicate with `all pods on all nodes` without NAT
    -   This model can be implemented using different Network plugins. Not all of them are covered in this list. To see more, visit [K8s documentation](https://v1-16.docs.kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-implement-the-kubernetes-networking-model)
        -   `Cisco ACI`
            -   integrated overlay and underlay SDN
            -   supports containers, virtual machines, and bare metal servers
        -   `Flannel`
            -   very simple overlay network
        -   `Project Calico`
            -   open source container networking provider and network policy engine
        -   `Canal`
            -   combination of two:
                -   `Calico` for networking
                -   `Flannel` for policy enforcement
        -   `Weave Net from Weaveworks`
            -   runs as a CNI plug-in or stand-alone
            -   doesn’t require any configuration or extra code to run
            -   the network provides one IP address per pod - as is standard for Kubernetes
---

## Understand service networking.
1.  `Pod <-> Service`
    -   utilizes [Services](./core-concepts.md#understand-services-and-other-network-primitives)
2.  `External <-> Service`
    -   utilizes [Services](./core-concepts.md#understand-services-and-other-network-primitives)
---

## Deploy and configure network load balancer.
### Create External LoadBalancer
-   ` Note: This feature is only available for cloud providers or environments which support external load balancers.`
-   Declarative
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: example-service
    spec:
      selector:
        app: example
      ports:
        - port: 8765
          targetPort: 9376
      type: LoadBalancer
    ```
-   Imperative
    ```bash
    kubectl expose deployment example \
    --port=8765 \
    --target-port=9376 \
    --name=example-service \
    --type=LoadBalancer
    ```
-   Finding LB IP address
    ```bash
    # using kubectl describe
    kubectl describe service example-service | grep "LoadBalancer Ingress"

    # using kubectl get (look for EXTERNAL-IP column)
    kubectl get service example service
    ```

### Configure kube-apiserver LoadBalancer
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
### Documentation materials:
-   [Create an External Load Balancer](https://v1-16.docs.kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/)
---

## Know how to use Ingress rules.
### What is Ingress ?
-   exposes HTTP / HTTPS routes from outside of the cluster towards services in cluster
-   traffic routing is controlled by `rules` defined on Ingress resource
-   exposing ports other than HTTP/HTTPS are not supported
    -   in order to do so, use `NodePort` or `LoadBalancer` services

### Pre-requisites
-   before one can use Ingress resources, an `Ingress controller` has to be deployed inside a cluster

### Ingress Resource
-   basic manifest example, which redirets all traffic coming to `/testpath` towards Service `test` and port `80`
    ```yaml
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: test-ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /
    spec:
      rules:
        - http:
          paths:
            - path: /testpath
              backend:
                serviceName: test
                servicePort: 80
    ```
-   Ingress resources are using `annotations` to control various configuration settings
    -   specific annotations supported depends on which Ingress controller had been deployed
-   Ingress rules
    -   `host`
        -   optional
        -   if not used, Ingress rule is routing all incoming traffic
        -   if used however, only traffic coming from specifiec host is being routed by the rule
    -   `paths`
        -   list of paths, which are considered when selecting traffic to route
        -   both `host` and `path` has to match the rule in order to be routed
        -   each `path` contains a `backend` associated with it
    -   `backend`
        -   definition of the destination where Ingress rule routes the traffic
        -   combination of `serviceName` and `servicePort` is exactly pointing to an existing service inside the cluster
            -   both `name` and `port` has to match with existing service
-   Default backend
    -   can be understood as a fallback rule (which does not have to be configured by Ingress objet)
    -   all traffic which is not caught by existing Ingress rules, is going to be redirected to Default backend (usually some 404 error page)
### Types of Ingress
-   Single Service Ingress
    -   can be exposed also by `NodePort` or `LoadBalancer` services
    -   in case of Ingress usage, manifest example is below:
        ```yaml
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        metadata:
          name: test-ingress
        spec:
          backend:
            serviceName: testsvc
            servicePort: 80
        ```
-   Simple fanout
    -   used when one needs to route traffic from single IP address towards more backend services (based on different paths for example)
        ```yaml
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        metadata:
          name: simple-fanout-example
          annotations:
            nginx.ingress.kubernetes.io/rewrite-target: /
        spec:
          rules:
          - host: foo.bar.com
            http:
              paths:
              - path: /foo
                backend:
                  serviceName: service1
                  servicePort: 4200
              - path: /bar
                backend:
                  serviceName: service2
                  servicePort: 8080
        ```
-   Name based virtual hosting
    -   combination of `Simple fanout` type, but for multiple hosts
    -   below manifest will route traffic:
        -   first.bar.com -> service1:80
        -   second.foo.com -> service2:80
        -   all others -> service3:80
        ```yaml
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        metadata:
          name: name-virtual-host-ingress
        spec:
          rules:
          - host: first.bar.com
            http:
            paths:
            - backend:
                serviceName: service1
                servicePort: 80
          - host: second.foo.com
            http:
            paths:
            - backend:
                serviceName: service2
                servicePort: 80
          - http:
            paths:
            - backend:
                serviceName: service3
                servicePort: 80
        ```
### TLS
-   Ingress can provide secured way in terms of HTTPS utilizing TLS
-   Ingress will terminate the TSL 
-   `Secret` object containing `tls.crt` and `tls.key` is required
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: testsecret-tls
      namespace: default
    data:
      tls.crt: base64 encoded cert
      tls.key: base64 encoded key
    type: kubernetes.io/tls
    ```
-   This secret object is then referred to inside the Ingress manifest
    -   if using below configuration, provided TLS certificate has to contain `CN=sslexample.foo.com` in order to work
    ```yaml
    apiVersion: networking.k8s.io/v1beta1
    kind: Ingress
    metadata:
      name: tls-example-ingress
    spec:
      tls:
      - hosts:
        - sslexample.foo.com
        secretName: testsecret-tls
      rules:
        - host: sslexample.foo.com
          http:
            paths:
            - path: /
            backend:
                serviceName: service1
                servicePort: 80
    ```

### Documentation materials:
-   [Ingress](https://v1-16.docs.kubernetes.io/docs/concepts/services-networking/ingress/)
-   [Ingress controllers](https://v1-16.docs.kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
---

## Know how to configure and use the cluster DNS.
### Introduction
-   K8s DNS component schedules a Pod + Service inside the cluster
-   Kubelets are then configured (by DNS component), so every container uses DNS Service's IP to resolve hostnames
-   Every service inside the cluster (including DNS service) is assigned with a hostname
-   By default, every Pod's DNS search list includes:
    -   Pod's own namespace
    -   cluster's default domain
    -   example:
        -   Service `service` is inside Namespace `namespace`
        -   Pod `pod` running inside `namespace` can lookup the service just by DNS query of `service`
        -   Pod `pod_far` running inside different namespace, has to use DNS query `service.namespace`
### Services
-   A records
    -   normal service
        -   assigned with `my-svc.my-namespace.svc.cluster-domain.example`
        -   resolves into clusterIP of the service
    -   headless service
        -   assigned with `my-svc.my-namespace.svc.cluster-domain.example`
        -   resolves into set of Pod's IPs selected by the service
-   SRV records
    -   for each named port, SRV record is created in a form
        -   `_my-port-name._my-port-protocol.my-svc.my-namespace.svc.cluster-domain.example`
    -   for normal service
        -   resolved into port number + domain name `my-svc.my-namespace.svc.cluster-domain.example`
    -   for headless service
        -   resolved into multiple results
        -   for each Pod backed by the service
            -   port number + domain name of the pod `auto-generated-name.my-svc.my-namespace.svc.cluster-domain.example`
### Pods
-   Pod’s hostname and subdomain fields
    -   default hostname of the pod refers to `.metadata.name` value
    -   if `.spec.hostname` is set, it will be used instead of the `.metadata.name`
    -   also `.spec.subdomain` can be specified
    -   if hostname is used, DNS will create A record for the Pod as well
    -   below Pod will have the FQDN:
        -   `busybox-1.default-subdomain.my-namespace.svc.cluster-domain.example`
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: busybox1
          labels:
            name: busybox
        spec:
          hostname: busybox-1
          subdomain: default-subdomain
          containers:
          - image: busybox:1.28
            command:
              - sleep
              - "3600"
            name: busybox
        ```
    -   Pod which is not in `Ready` state will not receive DNS hostname record
-   Pod’s DNS Policy
    -   configured in `.spec.dnsPolicy` of Pod
    -   `Default`
        -   The Pod inherits the name resolution configuration from the node that the pods run on
    -   `ClusterFirst`
        -   Resolution configuration depends on whether custom DNS configuration is applied
            -   without custom configuration
                -   same behavior as Default, inherited from Node where Pod runs
            -   with custom configuration
                -   Names with the cluster suffix -> coreDNS 
                -   Names with the stub domain suffix -> custom DNS resolver
                -   Names without a matching suffix -> Upstream DNS
    -   `ClusterFirstWithHostNet`
        -   should be set for Pods running with `.spec.hostNetwork: True`
    -   `None`
        -   ignores DNS settings from Kubernetes
        -   all DNS config is set by `.spec.dnsConfig` in Pods manifest
-   Pod’s DNS Config
    -   `nameservers`
        -   minimum 1 IP used as a DNS server in case of `.spec.dnsPolicy: None`
        -   maximum of 3 IPs
    -   `searches`
        -   a list of DNS search domains for hostname lookup in the Pod
    -   `options`
        -   an optional list of objects where each object may have `name`(required) and `value`
        -   contents in this property will be merged to the options generated from the specified DNS policy
-   Custom DNS configuration (coreDNS vs kubeDNS)
    -   kubeDNS
        -   example configMap to set custom DNS configuration
            ```yaml
            apiVersion: v1
            kind: ConfigMap
            data:
              stubDomains: |
                {"abc.com" : ["1.2.3.4"], "my.cluster.local" : ["2.3.4.5"]}
              upstreamNameservers: |
                ["8.8.8.8", "8.8.4.4"]
            ```
    -   coreDNS
        -   example of the same configuration, but converted to coreDNS
            ```yaml
            apiVersion: v1
            kind: ConfigMap
            metadata:
              name: coredns
              namespace: kube-system
            data:
              Corefile: /
                .:53 {
                        errors
                        health
                        kubernetes cluster.local in-addr.arpa ip6.arpa {
                            pods insecure
                            fallthrough in-addr.arpa ip6.arpa
                        }
                        prometheus :9153
                        forward .  8.8.8.8 8.8.4.4
                        cache 30
                    }
                    abc.com:53 {
                        errors
                        cache 30
                        forward . 1.2.3.4
                    }
                    my.cluster.local:53 {
                        errors
                        cache 30
                        forward . 2.3.4.5
                    }
            ```
### Documentation materials:
-   [DNS for Services and Pods](https://v1-16.docs.kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
-   [Customizing DNS Service](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)
---

## Understand CNI.
`Alpha in v1.15 -> subject to rapid changes`
### What is CNI ?
-   set of standards that define how programs (plugins) should be developed in order to solve networking challenges in CRE (Container Runtime Environments)
    -   container-runtime requirements
        -   creates network namespace
        -   identifies the network container has to attach to
        -   invokes Network Plugin (brigde) when container is added or deleted
        -   Network configuration in JSON format
    -   network plugin requirements
        -   must support ADD/DEL/CHECK command line args
        -   must support parameters like container id, network namespace, etc.
        -   must manage IP assignment to POD
        -   must return results in specific format
-   ultimate goal is a interchangeability, therefore any container runtime should be able to work with any network plugin, if both sides adhere to CNI standards

### Config details
-   Enable by starting `Kubelet` with argument `--network-plugin=cni`
-   Reads configuration from `/etc/cni/net.d` by default
-   any network plugin referenced in CNI config, must be present in folder specified in `--cni-bin-dir` (default to `/opt/cni/bin`)
-   example of config used in a cluster built by kubeadm v1.15.5 with calico network plugin used
    -   taken from `/etc/cni/net.d/10-canal.conflist`
    ```json
    {
    "name": "k8s-pod-network",
    "cniVersion": "0.3.1",
    "plugins": [
        {
        "type": "calico",
        "log_level": "info",
        "datastore_type": "kubernetes",
        "nodename": "master1",
        "ipam": {
            "type": "host-local",
            "subnet": "usePodCidr"
        },
        "policy": {
            "type": "k8s"
        },
        "kubernetes": {
            "kubeconfig": "/etc/cni/net.d/calico-kubeconfig"
        }
        },
        {
        "type": "portmap",
        "snat": true,
        "capabilities": {"portMappings": true}
        }
    ]
    }
    ``` 
### Fun facts
-   Docker itself does not support CNI standards
-   it has its own CNM (Container Network Model)
-   When Kubernetes is usind Docker as container runtime, every time it creates a container, it is done with `--network=none` and once the container is created, it then invokes CNI commands
### Documentation materials:
-   [Network Plugins](https://v1-16.docs.kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/)
-   Not usable during CKA (not in `kubernetes.io` domain)
    -   [CNI configuration](https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration)
    -   [Container Network Interface (CNI) Explained
](https://www.youtube.com/watch?v=l2BS_kuQxBA)
---