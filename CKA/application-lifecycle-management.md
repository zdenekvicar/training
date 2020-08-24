# Application Lifecycle Management

**Topics**
-   [Understand Deployments and how to perform rolling updates and rollbacks](#understand-deployments-and-how-to-perform-rolling-updates-and-rollbacks)
-   [Know various ways to configure applications](#know-various-ways-to-configure-applications)
-   [Know how to scale applications](#know-how-to-scale-applications)
-   [Understand the primitives necessary to create a self-healing application](#understand-the-primitives-necessary-to-create-a-self-healing-application)
---

## Understand Deployments and how to perform rolling updates and rollbacks.
```bash
# create deployment manifest
kubectl run nginx --image=nginx:1.15 --replicas=3 --dry-run -o yaml > deployment.yml

# create namespace
kubectl create ns nginx

# apply deployment manifest
kubectl apply -f deployment.yml -n nginx

# expose nginx
kubectl expose -n nginx deployment nginx --port=80

# check service
kubectl get service -n nginx

# test functionality (from cluster node)
curl <service_clusterIP>:80

# update deployment
vim deployment.yml # change image to nginx:1.16
kubectl apply -f deployment.yml

# check rollout status
kubectl -n nginx rollout status deployment nginx

# chech rollout history
kubectl -n nginx rollout history deployment nginx

# verify pod images
kubectl get pods -n nginx | grep Image:

# rollback
kubectl -n nginx rollout undo deployment nginx

# check progress 
kubectl -n nginx rollout status deployment nginx

# verify pod images
kubectl get pods -n nginx | grep Image:
```
## Know various ways to configure applications.
-   Use configmap
    -   used for non-sensitive information
    -   Create configmap
        -   declarative
            ```yaml
            apiVersion: v1
            data:
              exam.txt: |
                exam:CKA
            name: zdenekvicar
            kind: ConfigMap
            metadata:
              name: cmap
            ```
        -   imperative
            ```bash
            kubectl create configmap --from-literal=name=zdenekvicar --from-file=exam.txt cmap
            ```
    -   Use configmap (1 value)
        ```bash
        # apply example deployment to cluster
        cat <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          labels:
            run: nginx
          name: nginx
        spec:
          replicas: 1
          selector:
            matchLabels:
                run: nginx
          template:
            metadata:
                labels:
                  run: nginx
            spec:
              containers:
                - image: nginx:stable
                  name: nginx
                  env:
                    - name: name
                      valueFrom:
                        configMapKeyRef:
                          name: cmap
                          key: name
        EOF

        # verify functionality
        root@master1:~# kubectl exec nginx-547cbbc886-xd4hj -- env | grep name
        name=zdenekvicar
        ```
    -   Use configmap (all values)
        ```bash
        # apply example deployment to cluster
        cat <<EOF | kubectl apply -f -
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          labels:
            run: nginx
          name: nginx
        spec:
          replicas: 1
          selector:
            matchLabels:
                run: nginx
          template:
            metadata:
                labels:
                  run: nginx
            spec:
              containers:
                - image: nginx:stable
                  name: nginx
                  envFrom:
                    - configMapRef:
                        name: cmap
        EOF

        # verify functionality
        root@master1:~# kubectl exec nginx-58d6565db-hlrwr -- env | egrep 'name|exam.txt'
        exam.txt=exam:CKA
        name=zdenekvicar
        ``` 
-   Use secret
    -   usage is almost the same as in case of configMaps
    -   only difference is with Secret itself, it has to contain base64 encoded values
## Know how to scale applications.
-   imperative
    -   `kubectl scale deployment <name> --replicas=x`
-   declarative
    -   adjust `.spec.replicas` to required value in manifest + `kubectl apply`
    -   also possible with `kubectl edit deployment <name>`
## Understand the primitives necessary to create a self-healing application.
-   ReplicaSet / Deployments
    -   self-healing is achieved by control loops watching the desired state of replicas
    -   if a replica fails, new one is started immediately
-   Probes
    -   support 3 various checks
        -   command
        -   http
        -   tcp get
    -   Liveness probe
        -   in case of check failure, container is restarted
    -   Readiness probe
        -   in case of check failure, Pod is removed from service (marked as NotReady as well)
    -   example with probes
        -   Liveness
            -   checking availability of TCP port 8080
            -   checks starts 15 seconds after container started
            -   check is every 20 seconds
        -   Readiness
            -   checking availability of TCP port 8080
            -   checks starts 5 sedconds after container started
            -   check is every 10 seconds
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: goproxy
          labels:
            app: goproxy
        spec:
          containers:
          - name: goproxy
            image: k8s.gcr.io/goproxy:0.1
            ports:
            - containerPort: 8080
            readinessProbe:
              tcpSocket:
                port: 8080
              initialDelaySeconds: 5
              periodSeconds: 10
            livenessProbe:
              tcpSocket:
                port: 8080
              initialDelaySeconds: 15
              periodSeconds: 20
        ```