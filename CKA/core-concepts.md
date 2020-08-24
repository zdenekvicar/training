# Core Concepts

**Topics**
-   [Understand the Kubernetes API primitives](#understand-the-kubernetes-api-primitives)
-   [Understand the Kubernetes cluster architecture](#understand-the-kubernetes-cluster-architecture)
-   [Understand Services and other network primitives](#understand-services-and-other-network-primitives)
---

## Understand the Kubernetes API primitives.
Kubernetes API is exposed by `api-server` - one of the main master components in k8s architecture.

### API versioning
1.  **Stable**
    -   name: vX -> i.e. `v1`
2.  **Alpha**
    -   name: contains alpha -> i.e. `v1alpha1`
    -   may be buggy
    -   disabled by default
    -   possible drop of support at any time
    -   only for short-lived testing cluster
    -   `NOT FIT FOR PRODUCTION`
    
3.  **Beta**
    -   name: contains beta -> i.e. `v2beta3`
    -   enabled by default
    -   considered safe to use (well tested)
    -   support will not be dropped, however some details might change over time
    -   recommended `for non-business-critical only` 

### API groups
API objects are separated into groups:
1.  **core group**
    -   also known as legacy group
    -   REST path: `api/v1`
    -   uses `apiVersion: v1`
2.  **named groups**
    -   several groups here -> full list in [API reference v1.16](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/)
    -   REST path: `/apis/$GROUP_NAME/$VERSION`
    -   uses `apiVersion: $GROUP_NAME/$VERSION` -> `apiVersion: batch/v1` or `apps/v1`
    -   can be extended by usage of `CustomResourceDefinition` or also known as `CRD`

### Enabling API groups
-   driven by `--runtime-config=` setting on `api-server`
-   accepts comma separated list
-   examples:
    -   disable `batch/v1` -> `--runtime-config=batch/v1=false`
    -   enable `batch/v2alpha1` -> `--runtime-config=batch/v2alpha1` 
-   enabling or disabling groups requires restart of `api-server` in order to load new runtime-config settings

### Enabling resources in the groups
-   even when we enable some API group, some of its resources might be still disabled by default
-   example: `extensions/v1beta1` does have following resources enabled by default:
    -   DaemonSets
    -   Deployments
    -   HorizontalPodAutoscalers
    -   Ingresses
    -   Jobs
    -   ReplicaSets
-   enable / disable is again driven with `runtime-config` setting on `api-server`
    -   example: `--runtime-config=extensions/v1beta1/deployments=false,extensions/v1beta1/ingresses=false`

### Documentation materials 
-   [The Kubernetes API](https://v1-16.docs.kubernetes.io/docs/concepts/overview/kubernetes-api/)
-   [Understanding Kubernetes Objects](https://v1-16.docs.kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)
-   [Kubernetes Object Management](https://v1-16.docs.kubernetes.io/docs/concepts/overview/working-with-objects/object-management/)

---
## Understand the Kubernetes cluster architecture.

### Master Components a.k.a control plane
Running mainly on master nodes
-   **`kube-apiserver`**
    -   exposes Kubernetes API, therefore enables communication to and from API itself
    -   scaled horizontally -> we can run more replicas of apiserver
-   **`etcd`**
    -   key:value store DB
    -   contains all cluster data (every object, etc.)
    -   losing etcd database equals losing K8s cluster
-   **`kube-scheduler`**
    -   watches API for newly created Pods without a node assigned to them
    -   selects viable node for any unassigned Pod found
    -   considers many factors while selecting viable node
        -   individual and collective resource requirements
        -   hardware/software/policy constraints
        -   affinity and anti-affinity specifications
        -   data locality
        -   inter-workload interference
        -   deadlines
-   **`kube-controller-manager`**
    -   responsible for running controllers inside cluster, including
        -   `Node controller`: watches and notifies when node goes down
        -   `Replication controller`: maintains correct number of replicas for all replication controller objects in cluster 
        -   `Endpoints controller`: creates endpoint objects (which connects Pods and Services together)
        -   `Service accounts & token controllers`: creates default accounts and API access tokens for each new namespaces
-   **`cloud-controller-manager`**
    -   runs controllers responsible for interaction with cloud providers
    -   cloud-controller-manager binary in `alpha` feature since K8s v1.6
    -   contains:
        -   `Node controller`: checks with cloud provider whether unresponsive node has been deleted in the cloud
        -   `Route controller`: responsible for setting routes in underlying cloud infrastructure
        -   `Service controller`: creates, updates, deletes cloud provider LoadBalancers
        -   `Volume controller`: creates, attaches, mounts, interacts with cloud provider to orchestrate volumes

### Node Components
Runs on every node, responsible for maintaining pods and providing Kubernetes runtime environment.
-   **`kubelet`**
    -   agent running on every node in cluster
    -   receives `PodSpecs` provided through various mechanisms and ensures containers described in these specs are running & healthy   
    -   ignores containers not created by Kubernetes
-   **`kube-proxy`**
    -   network proxy, running on every node in cluster
    -   maintains network rules on nodes, allowing network communication to K8s Pods
    -   if OS packet filtering layer is available, then kube-proxy utilizes it
    -   if no OS packet filtering layer is present, kube-proxy forwards the traffic on its own
-   **`container runtime`**
    -   software responsible for running containers
    -   K8s supports several of them:
        -   Docker
        -   containerd
        -   cri-o
        -   rktlet
        -   any implementation of Kubernetes CRI (Container Runtime Interface)

### Add-ons
Uses K8s objects to implement cluster-wide feature, like DNS.
-   **`DNS`**
    -   While the other addons are not strictly required, all Kubernetes clusters should have cluster DNS, as many examples rely on it.
    -   Cluster DNS is a DNS server, in addition to the other DNS server(s) in your environment, which serves DNS records for Kubernetes services.
    -   Containers started by Kubernetes automatically include this DNS server in their DNS searches.

### Documentation materials
-   [Kubernetes Components](https://v1-16.docs.kubernetes.io/docs/concepts/overview/components/)

---

## Understand Services and other network primitives.

### Services
-   abstraction which sets a policy how to access a logical set of Pods
-   **`Service with selector`**
    -   most common service type, utilizes selectors to define logical set of Pods
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: my-service
        spec:
          selector:
            app: MyApp
          ports:
            - protocol: TCP
              port: 80
              targetPort: 9376
        ```
-   **`Service without selector`**
    -   used when you need to define the Service target manually, instead of K8s auto-mapping
        ```yaml
        apiVersion: v1
        kind: Service
        metadata:
          name: my-service
        spec:
          ports:
            - protocol: TCP
              port: 80
              targetPort: 9376
        ```
    -   in such scenarios, `Endpoints` object has to be created manually
        ```yaml
        apiVersion: v1
        kind: Endpoints
        metadata:
          name: my-service
        subsets:
          - addresses:
              - ip: 192.0.2.42
            ports:
              - port: 9376
        ```
-   **`Multi port service`**
    -   requires `spec.ports.name` values
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: my-service
    spec:
      selector:
        app: MyApp
      ports:
        - name: http
          protocol: TCP
          port: 80
          targetPort: 9376
        - name: https
          protocol: TCP
          port: 443
          targetPort: 9377
      ```
-   **`Headless service`**
    -   created with `.spec.clusterIP` to `"None"`
    -   kube-proxy does not handle these Services
    -   no load-balancing or proxying is done by the K8s platorm

### Service Types
-   **`ClusterIP`**
    -   Exposes the Service on a cluster-internal IP
    -   Service only reachable from within the cluster
    -   default `ServiceType`
-   **`NodePort`**
    -   Exposes the Service on each Node’s IP at a static port (the NodePort)
    -   A ClusterIP Service, to which the NodePort Service routes, is automatically created
    -   Service reachable from outside the cluster, by requesting `NodeIP:NodePort`
        -   User -> NodePort Service -> ClusterIP Service -> Endpoint
-   **`LoadBalancer`**
    -   Exposes the Service externally using a cloud provider’s load balancer
    -   NodePort and ClusterIP Services, to which the external load balancer routes, are automatically created
        -   User -> LoadBalancer -> NodePort Service -> ClusterIP Service -> Endpoint
-   **`ExternalName`**
    -   CoreDNS version 1.7 or higher required to use this type
    -   Maps the Service to the contents of the `externalName` field (e.g. foo.bar.example.com), by returning a CNAME record with its value
    -   No proxying of any kind is set up

### kube-proxy 

Every node in a Kubernetes cluster runs a kube-proxy. kube-proxy is responsible for implementing a form of virtual IP for Services of type other than ExternalName.

-   **`User space proxy mode`**
    -   kube-proxy watches the Kubernetes `master` for the addition and removal of Service and Endpoint objects
    -   For each Service it opens a port (randomly chosen) on the local node
    -   takes the `SessionAffinity` setting of the Service into account when deciding which backend Pod to use
    -   user-space proxy installs iptables rules which capture traffic to the Service’s clusterIP (which is virtual) and port
    -   By default, kube-proxy in userspace mode chooses a backend via a `round-robin algorithm`
    -   retry mechanism -> user-space proxy can detect connection to failed Pod and retry with different backend Pod
-   **`iptables proxy mode`**
    -   kube-proxy watches the Kubernetes `control plane` for the addition and removal of Service and Endpoint objects
    -   For each Service, it installs iptables rules, which capture traffic to the Service’s clusterIP and port
    -   For each Endpoint object, it installs iptables rules which select a backend Pod
    -   By default, kube-proxy in iptables mode chooses a backend at `random`
    -   lower system overhead -> traffic is handled by Linux netfilter without the need to switch between userspace and the kernel space
    -   likely to be more reliable
    -   no retry mechanism -> connection to failed Pod is not detected in advance and therefore failed -> `Readiness probe` should be used to detect failed Pod
-   **`IPVS proxy mode`**
    -   kube-proxy watches `Kubernetes Services and Endpoints`, calls netlink interface to create IPVS rules accordingly and synchronizes IPVS rules with Kubernetes Services and Endpoints periodically
    -   When accessing a Service, IPVS directs traffic to one of the backend Pods
    -   The IPVS proxy mode is based on netfilter hook function that is similar to iptables mode, but uses hash table as the underlying data structure and works in the kernel space. That means kube-proxy in IPVS mode redirects traffic with a `lower latency` than kube-proxy in iptables mode, with `much better performance` when synchronising proxy rules. Compared to the other proxy modes, IPVS mode also supports a `higher throughput of network traffic`.
    -   multiple options for balancing traffic:
        -   `rr` - Round Robin
        -   `lc` - least connection (picking the endpoint with lowest connections)
        -   `sed` - shortest expected delay
        -   etc...
    -   To run kube-proxy in IPVS mode, you must make the IPVS Linux available on the node before you starting kube-proxy.
    -   If IPVS Linux is not found, kube-proxy falls back to iptables mode

### Service discovery
1.  **`Environment variables`**
    -   kubelet adds environment variables of each active Service after starting a Pod
    -   example: `{SVCNAME}_SERVICE_PORT` and `{SVCNAME}_SERVICE_HOST`
    -   `NOTE:` if using this method, Service has to be create before a Pod that needs to access it (otherwise env vars are not created by kubelet in start time of Pod, because Service is not yet active)
2.  **`DNS`**
    -   recommended option
    -   A cluster-aware DNS server, such as CoreDNS, watches the Kubernetes API for new Services and creates a set of DNS records for each one. 
    -   If DNS has been enabled throughout your cluster then all Pods should automatically be able to resolve Services by their DNS name
### Documentation materials
-   [Service](https://v1-16.docs.kubernetes.io/docs/concepts/services-networking/service/)

---
