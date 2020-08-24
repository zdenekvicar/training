# Security

**Topics**
-   [Know how to configure authentication and authorization](#know-how-to-configure-authentication-and-authorization)
-   [Understand Kubernetes security primitives](#understand-kubernetes-security-primitives)
-   [Know to configure network policies](#know-to-configure-network-policies)
-   [Create and manage TLS certificates for cluster components](#create-and-manage-tls-certificates-for-cluster-components)
-   [Work with images securely](#work-with-images-securely)
-   [Define security contexts](#define-security-contexts)
-   [Secure persistent key value store](#secure-persistent-key-value-store)
---

## Know how to configure authentication and authorization.
### 1. Authentication
Before you can start using Kubernetes, you have to be authenticated or logged in ...
-   **`Users in Kubernetes`**
    -   Normal users
        -   Not managed by Kubernetes -> managed by outside, indepentent service
    -   Service accounts
        -   Managed by Kubernetes
        -   Bound to specific namespace
        -   Created automatically by API server or manually through API call
        -   Tied to set of credentials stored as `Secrets`
-   **`Authentication strategies`**
    -   Kubernetes supports a lot of different Auth strategies, explanation of each with a guide how to use it is to be found by clicking on each of the below strategy type. However one is going to be explained here as well as it is the default one -> `Service Account Tokens`
    -   [X509 Client Certs](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/#x509-client-certs)
    -   [Static Token File](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/#static-token-file)
    -   [Bootstrap Tokens](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/#bootstrap-tokens)
    -   [Static Password File](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/#static-password-file)
    -   [Service Account Tokens](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens)
        -   A service account is an automatically enabled authenticator that uses signed bearer tokens to verify requests.
        -   Usually created automatically by API server
        -   Associated with pods through the `ServiceAccount` Admission Controller
        -   Bearer tokens are then mounted into Pods, allowing them to talk with API
        -   Account can be bound to Pod by `serviceAccountName` under `PodSpec`
        -   Create service account
            -   `kubectl create serviceaccount <name>`
        -   Check secret associated with account
            -   `kubectl get serviceaccount <name> -o yaml` and look for `.secrets.name`
            -   Secret contains public CA of apiserver + signed Json Web Token (JWT), which can be used to authenticate against API server
            -   `kubectl get secret <name> -o yaml` and look under `.data` for `ca.crt` and `token`
    -   [OpenID Connect Tokens](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens)
    -   [Webhook Token Authentication](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication)
    -   [Authenticating Proxy](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/#authenticating-proxy)

### 2. Authorization
Once user is logged in (or authenticated), Kubernetes is capable of running an Authorization checks whether a specific user has permissions for requested objects and actions. K8s offers again multiple choises, `RBAC` mode is going to be explaned more below.
-   [Node](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/node/)
    -   mode that grants permissions to kubelets based on the pods they are scheduled to run
-   [ABAC](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/abac/)
    -   Attribute-based access control (ABAC) defines an access control paradigm whereby access rights are granted to users through the use of policies which combine attributes together
-   [Webhook](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/webhook/)
    -   A WebHook is an HTTP callback, an simple HTTP POST event-notification that occurs when something happens
-   [RBAC](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/)
    -   Role-based access control (RBAC) is a method of regulating access to computer or network resources based on the roles of individual users within an enterprise
    -   `Role` contains rules that represents sets of permissions
        -   Permissions are additive only (there are no `deny` rules)
    -   `RoleBinding` grants the permissions defined by a role to user or set of users
    -   Namespace wide -> `Role` + `RoleBinding`
        ```yaml
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: Role
        metadata:
          namespace: default
          name: pod-reader
        rules:
        - apiGroups: [""] # "" indicates the core API group
          resources: ["pods"]
          verbs: ["get", "watch", "list"]

        ---
        apiVersion: rbac.authorization.k8s.io/v1
        # This role binding allows "jane" to read pods in the "default" namespace.
        kind: RoleBinding
        metadata:
          name: read-pods
          namespace: default
        subjects:
        - kind: User
          name: jane # Name is case sensitive
          apiGroup: rbac.authorization.k8s.io
        roleRef:
          kind: Role #this must be Role or ClusterRole
          name: pod-reader # this must match the name of the Role or ClusterRole you wish to bind to
          apiGroup: rbac.authorization.k8s.io
        ```
    -   Cluster wide -> `ClusterRole` + `ClusterRoleBinding`
        ```yaml
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRole
        metadata:
          # "namespace" omitted since ClusterRoles are not namespaced
          name: secret-reader
        rules:
        - apiGroups: [""]
          resources: ["secrets"]
          verbs: ["get", "watch", "list"]

        ---
        apiVersion: rbac.authorization.k8s.io/v1
        # This cluster role binding allows anyone in the "manager" group to read secrets in any namespace.
        kind: ClusterRoleBinding
        metadata:
          name: read-secrets-global
        subjects:
        - kind: Group
          name: manager # Name is case sensitive
          apiGroup: rbac.authorization.k8s.io
        roleRef:
          kind: ClusterRole
          name: secret-reader
          apiGroup: rbac.authorization.k8s.io
        ```


### Documentation materials 
-   [Authenticating](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/)
-   [Authorization Overview](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/authorization/)
-   [Using RBAC Authorization](https://v1-16.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/)

---
## Understand Kubernetes security primitives.
### Components of the Cluster
-   [Controlling access to the Kubernetes API](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/#controlling-access-to-the-kubernetes-api)
    -   Use Transport Layer Security (TLS) for all API traffic
    -   API Authentication
    -   API Authorization
-   [Controlling access to the Kubelet](https://v1-16.docs.kubernetes.io/docs/admin/kubelet-authentication-authorization)
    -   Kubelet authentication
    -   Kubelet authorization
-   [Controlling the capabilities of a workload or user at runtime](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/#controlling-the-capabilities-of-a-workload-or-user-at-runtime)
    -   Limiting resource usage on a cluster
        -   ResourceQuota
        -   LimitRanges
    -   Controlling what privileges containers run with
        -   Utilizes `Pod Security Policy`
    -   Restricting network access
        -   Utilizes `Network Policy`
    -   Restricting cloud metadata API access
-   [Protecting cluster components from compromise](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/#protecting-cluster-components-from-compromise)
    -   Restrict access to etcd
    -   Enable [audit](https://v1-16.docs.kubernetes.io/docs/tasks/debug-application-cluster/audit/) logging
    -   Restrict access to alpha or beta features
    -   Rotate infrastructure credentials frequently
    -   Review third party integrations before enabling them
    -   Encrypt secrets at rest
    -   Receiving alerts for security updates and reporting vulnerabilities

### Components in the Cluster (your application)
-   [List of areas & recommendations](https://v1-16.docs.kubernetes.io/docs/concepts/security/overview/#components-in-the-cluster-your-application)
    -   RBAC Authorization (Access to the Kubernetes API)	
    -   Authentication
    -   Application secrets management (and encrypting them in etcd at rest)	
    -   Pod Security Policies	
    -   Quality of Service (and Cluster resource management)	
    -   Network Policies	
    -   TLS For Kubernetes Ingress	

### Documentation materials 
-   [Securing a Cluster](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/securing-a-cluster/)
-   [Overview of Cloud Native Security](https://v1-16.docs.kubernetes.io/docs/concepts/security/overview/)

---
## Know to configure network policies.
-   Create an nginx deployment and expose it via a service
    ```bash
    kubectl run nginx --image=nginx --replicas=2
    kubectl expose deployment nginx --port=80
    kubectl get svc,pod
    ```
-   Test the service by accessing it from another pod
    ```bash
    kubectl run busybox --rm -ti --image=busybox /bin/sh
    # inside the busybox container
    wget --spider --timeout=1 nginx
    ```
-   Limit access to the nginx service
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: NetworkPolicy
    metadata:
      name: access-nginx
    spec:
      podSelector:
        matchLabels:
          run: nginx
      ingress:
      - from:
        - podSelector:
            matchLabels:
              access: "true"
    ```
    ```bash
    kubectl apply -f nginx-policy.yaml
    ```
-   Test access to the service when access label is not defined
    ```bash
    kubectl run busybox --rm -ti --image=busybox /bin/sh
    # inside the busybox container
    wget --spider --timeout=1 nginx
    ```
-   Define access label and test again
    ```bash
    kubectl run busybox --rm -ti --labels="access=true" --image=busybox /bin/sh
    # inside the busybox container
    wget --spider --timeout=1 nginx
    ```

### Documentation materials  
-   [Declare Network Policy](https://v1-16.docs.kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/)
-   [Network Policies](https://v1-16.docs.kubernetes.io/docs/concepts/services-networking/network-policies/)

---
## Create and manage TLS certificates for cluster components.

### Installing CFSSL tool
```bash
apt install golang-cfssl
```

### Create CSR (Certificate Signing Request)
Below command creates 2 files:
- `server.csr`: PEM encoded pkcs#10 certification request
- `server-key.pem`: PEM encoded key to the certificate that is still to be created
```bash
cat <<EOF | cfssl genkey - | cfssljson -bare server
{
  "hosts": [
    "{service_name}.{namespace}.svc.cluster.local",
    "{pod_name}.{namespace}.pod.cluster.local",
    "{service_clusterIP}",
    "{POD_IP}"
  ],
  "CN": "{pod_name}.{namespace}.pod.cluster.local",
  "key": {
    "algo": "ecdsa",
    "size": 256
  }
}
EOF
```

### Create CSR K8s object manifest
```bash
# creates the manifest
cat > csr.yml <<EOF
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: {service_name}.{namespace}
spec:
  request: $(cat server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF

# apply manifest to cluster
kubectl apply -f csr.yml

# verify CSR existence
kubectl get csr
kubectl describe csr {service_name}.{namespace}
```

### Approving / Rejecting CSR manually
```bash
# approve CSR 
kubectl certificate approve {service_name}.{namespace}
```

### Obtaining the certificate file
With below commant, you can obtain PEM encoded certificate `server.crt`
```bash
kubectl get csr {service_name}.{namespace} -o jsonpath='{.status.certificate}' | base64 --decode > server.crt
```

### Documentation materials  
-   [Manage TLS Certificates in a Cluster](https://v1-16.docs.kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)

---
## Work with images securely.
### imagePullPolicy
  - `Always`
    - ensures any Pod has to download fresh image before it can start the container
    - this setup will ignore any pre-pulled images on Node itself
    - can be used in combination with some secured images, where we need to be sure only Pods with proper credentials can download & run such image
  - `IfNotPresent`
    - the image is pulled only if it is not already present locally
  - `Never`
    - the image is assumed to exist locally
    - no attempt is made to pull the image
  - no imagePullPolicy + `latest` tag
    - `Always` is applied
  - no imagePullPolicy + no latest tag
    - `IfNotPresent` is applied

### imagePullSecrets
  - docker credentials can be put into a K8s secret, which can be refered to by any configured POD
    ```bash
    # create secret
    kubectl create secret docker-registry <name> \
     --docker-server=DOCKER_REGISTRY_SERVER \
     --docker-username=DOCKER_USER \
     --docker-password=DOCKER_PASSWORD \
     --docker-email=DOCKER_EMAIL
    ```
  - such secret can be assigned to a pod using `.spec.imagePullSecrets`

### general recommendations
- `latest` image tag usage should be avoided when deploying containers in production as it is harder to track which version of the image is running and more difficult to roll back properly.
- Container Vulnerability Scanning and OS Dependency Security	
  - containers should be periodically scanned by some scanner tool (Clair project)
- Image Signing and Enforcement	
  - containers should be signed (Notary project) and only confirmed signed images should be allowed to enter the cluster (Portieris project)
- Disallow privileged users	
  - users used to run applications inside the containers should have the least level of privilege required to run the application only
  - we should avoid giving bigger privileges than the least required ones

### Documentation materials  
- [Images](https://v1-16.docs.kubernetes.io/docs/concepts/containers/images/)
- [Configuration Best Practices - Container Images](https://v1-16.docs.kubernetes.io/docs/concepts/configuration/overview/#container-images)
- [Overview of Cloud Native Security - Container](https://v1-16.docs.kubernetes.io/docs/concepts/security/overview/#container)

---
## Define security contexts.
A security context defines privilege and access control settings for a Pod or Container

### Set the security context for a Pod
- configured inside `.spec.securityContext` of a Pod manifest
- is applied to every container inside the pod
- all configuration options can be found in API reference for [PodSecurityContext object](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#podsecuritycontext-v1-core)
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: security-context-demo
  spec:
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      fsGroup: 2000
    volumes:
    - name: sec-ctx-vol
      emptyDir: {}
    containers:
    - name: sec-ctx-demo
      image: busybox
      command: [ "sh", "-c", "sleep 1h" ]
      volumeMounts:
      - name: sec-ctx-vol
        mountPath: /data/demo
      securityContext:
        allowPrivilegeEscalation: false
  ```

### Set the security context for a Container
- configured inside `.spec.containers.securityContext` of a Pod manifest
- affects only specific container
- override settings applied on PodSecurityContext level
- does not apply for Pod volumes
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: security-context-demo-2
  spec:
    securityContext:
      runAsUser: 1000
    containers:
    - name: sec-ctx-demo-2
      image: gcr.io/google-samples/node-hello:1.0
      securityContext:
        runAsUser: 2000
        allowPrivilegeEscalation: false
  ```

### Set capabilities for a Container
- With Linux capabilities, you can grant certain privileges to a process without granting all the privileges of the root user.
- configured inside `.spec.containers.securityContext.capabilities`
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: security-context-demo-4
  spec:
    containers:
    - name: sec-ctx-4
      image: gcr.io/google-samples/node-hello:1.0
      securityContext:
        capabilities:
          add: ["NET_ADMIN", "SYS_TIME"]
  ```

### Assign SELinux labels to a Container
- configured inside:
  - `spec.securityContext.seLinuxOptions` for Pods
  - `spec.containers.securityContext.seLinuxOptions` for Containers
- API reference for [SELinuxOptions object](https://v1-16.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#selinuxoptions-v1-core)
  ```yaml
  ...
  securityContext:
    seLinuxOptions:
      level: "s0:c123,c456"
  ```

### Documentation materials  
- [Configure a Security Context for a Pod or Container
](https://v1-16.docs.kubernetes.io/docs/tasks/configure-pod-container/security-context/)

---
## Secure persistent key value store.

### Documentation materials  
- [Secret](https://v1-16.docs.kubernetes.io/docs/concepts/configuration/secret/)