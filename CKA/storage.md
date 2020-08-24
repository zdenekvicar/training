# Storage

**Topics**
-   [Understand persistent volumes and know how to create them](#understand-persistent-volumes-and-know-how-to-create-them)
-   [Understand access modes for volumes](#understand-access-modes-for-volumes)
-   [Understand persistent volume claims primitive](#understand-persistent-volume-claims-primitive)
-   [Understand Kubernetes storage objects](#understand-kubernetes-storage-objects)
-   [Know how to configure applications with persistent storage](#know-how-to-configure-applications-with-persistent-storage)
---

## Understand persistent volumes and know how to create them.
### Introduction
-   cluster-wide resource
-   created manually -> PV object created by K8s cluster admin
-   created automatically -> `StorageClass`
-   PV provides a 'link' to physical storage
    -   `Physical storage` -> `PV` -> `PVC` -> `POD` -> `Container`
-   Storage Object in use Protection
    -   feature enabled by default
    -   its function is to ensure actively used PV & PVC objects are not deleted -> prevents loss of data
    -   active use = there is a Pod object which uses the PVC 
    -   proper deletion flow:
        -   `Pod` -> `PVC` -> `PV`
-   Reclaim policy
    -   `Retain`
        -   if PVC is removed from cluster, PV object retains in `Released` state
        -   this state does not allow reusage of the PV (as it still has ClaimRef filled)
        -   reason: allow K8s cluster admin to backup the data before removing the PV
    -   `Delete`
        -   deletes not only PV, but also storage under it
        -   has to have support via Plugin
        -   supported in cloud environments (AWS, Azure, ...)
    -   `Recycle`
        -   `deprecated` -> recommendation is to use dynamic provisioning (combination of Delete + StorageClass)
        -   if supported by plugin, reclaim policy performs `rm -rf /volume/*` and makes it available for another usage
-   Example of PV object backed by NFS storage
    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: pv0003
    spec:
      capacity:
        storage: 5Gi
      volumeMode: Filesystem
      accessModes:
        - ReadWriteOnce
      persistentVolumeReclaimPolicy: Recycle
      storageClassName: slow
      mountOptions:
        - hard
        - nfsvers=4.1
      nfs:
        path: /tmp
        server: 172.17.0.2
    ```

### Documentation materials:
-   [Persistent Volumes](https://v1-16.docs.kubernetes.io/docs/concepts/storage/persistent-volumes/)
---

## Understand access modes for volumes.
There are 3 different modes available:
-   `ReadWriteOnce`
    -   abbreviation: `RWO`
    -   the volume can be mounted as read-write by a single node
    -   supported by all volume types
-   `ReadOnlyMany`
    -   abbreviation: `ROX`
    -   the volume can be mounted read-only by many nodes
    -   supported only by some selected volume types:
        -   AzureFile, CephFS, FC, ...
-   `ReadWriteMany`
    -   abbreviation: `RWX`
    -   the volume can be mounted as read-write by many nodes
    -   supported only by some selected volume types:
        -   Azurefile, CephFS, Glusterfs, NFS, ...

### Documentation materials:
-   [Access Modes](https://v1-16.docs.kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)
---

## Understand persistent volume claims primitive.
-   PersistentVolumeClaim (PVC) object provides a way how to use existing Persistent Volume (PV) in a Pod
-   PVC will bind only in case it finds valid PV based on its criterias:
    -   accessModes
    -   volumeMode
    -   resources.requests.storage
    -   selector
        -   matchLabels
        -   matchExpressions
-   If no valid PV is found, PVC remains in Pending state and Pod using such PVC cannot be started
-   Example manifest
    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: myclaim
    spec:
      accessModes:
        - ReadWriteOnce
      volumeMode: Filesystem
      resources:
        requests:
          storage: 8Gi
      storageClassName: slow
      selector:
        matchLabels:
          release: "stable"
        matchExpressions:
          - {key: environment, operator: In, values: [dev]}
    ```
### Documentation materials:
-   [PersistentVolumeClaims](https://v1-16.docs.kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims)

---

## Understand Kubernetes storage objects.
Kuberentes provides many types of Volumes, some of them are only temporary and shares the lifecycle with Pod, while others provide full persistence even in case of Pod death: 
-   `hostPath`
    -   volume mounts a file or directory from the host node’s filesystem into your Pod
    -   can be dangerous in case of multi-node cluster, where Pod can be re-scheduled on a different node, therefore loosing data
    -   one can define `type` parameter in a manifest:
        -   `DirectoryOrCreate`
            -   If nothing exists at the given path, an empty directory will be created there as needed with permission set to 0755, having the same group and ownership with Kubelet.
        -   `Directory`
            -   A directory must exist at the given path
        -   `FileOrCreate`
            -   If nothing exists at the given path, an empty file will be created there as needed with permission set to 0644, having the same group and ownership with Kubelet.
        -   `File`
            -   A file must exist at the given path
        -   `Socket`
            -   A UNIX socket must exist at the given path
        -   `CharDevice`
            -   A character device must exist at the given path
        -   `BlockDevice`
            -   A block device must exist at the given path
    -   example Pod manifest using hostPath:
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: test-pd
        spec:
          containers:
          - image: k8s.gcr.io/test-webserver
            name: test-container
            volumeMounts:
              - mountPath: /test-pd
                name: test-volume
        volumes:
          - name: test-volume
            hostPath:
              path: /data
              type: Directory
        ```
-   `local`
    -   very similar to hostPath
    -   does require manual creation of PV
    -   contains affinity settings to match specific Node
        ```yaml
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: example-pv
        spec:
          capacity:
            storage: 100Gi
          volumeMode: Filesystem
          accessModes:
            - ReadWriteOnce
          persistentVolumeReclaimPolicy: Delete
          storageClassName: local-storage
          local:
            path: /mnt/disks/ssd1
          nodeAffinity:
            required:
              nodeSelectorTerms:
                - matchExpressions:
                  - key: kubernetes.io/hostname
                    operator: In
                    values:
                      - example-node
        ```
-   `emptyDir`
    -   created empty
    -   created as soon as the Pod is scheduled on a specific Node
    -   shares lifecycle with Pod = does not provide persistence
    -   however survives Container crashes, since emptyDir life is tied only to Pod itself
    -   by default, emptyDir uses whatever storage medium is available (SDD, HDD, ...)
    -   can be pointed to use Memory instead -> `epmtyDir.medium: "Memory"`
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: test-pd
        spec:
          containers:
            - image: k8s.gcr.io/test-webserver
              name: test-container
              volumeMounts:
                - mountPath: /cache
                  name: cache-volume
        volumes:
          - name: cache-volume
            emptyDir: {}
        ```
-   `configMap` / `secret`
    -   both objects can be mounted to a Pod as a Volume, therefore they are also part of this list
    -   however, both of them provides just ReadOnly option -> Pod cannot write inside configMap or secret objects when they are mounted as volumes
    -   mostly used to mount some configuration files, or passwords inside a Pod
-   `nfs`
    -   allows an existing NFS (Network File System) share to be mounted into your Pod
    -   provides persistence -> survives death of Pod
    -   supports RWX mode
-   `azureFile` / `azureDisk`
    -   both are Azure cloud specific options
    -   azureFile
        -   supports RWX mode
        -   utilizes azure StorageAccount
    -   azureDisk
        -   utilizes azure Data Disk
-   ... (for full list of files, navigate on the link in Documentation materials section below)
### Documentation materials:
-   [Volumes](https://v1-16.docs.kubernetes.io/docs/concepts/storage/volumes/)

---

## Know how to configure applications with persistent storage.
-   EmptyDir example:
    -   does not require PV nor PVC to be used
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: redis
    spec:
      containers:
      - name: redis
        image: redis
        volumeMounts:
        - name: redis-storage
          mountPath: /data/redis
    volumes:
    - name: redis-storage
        emptyDir: {}
    ```
-   Most volumes however requires usage of PV and PVC to be used
    -   AzureFile example:
        -   consider all Azure prerequisites are already created (StorageAccount with FileShare)
    -   Create RBAC rule to allow AzureFile usage
        ```yaml
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRole
        metadata:
          name: system:azure-cloud-provider
        rules:
          - apiGroups: ['']
            resources: ['secrets']
            verbs:     ['get','create']
        
        ---
        apiVersion: rbac.authorization.k8s.io/v1
        kind: ClusterRoleBinding
        metadata:
          name: system:azure-cloud-provider
        roleRef:
          kind: ClusterRole
          apiGroup: rbac.authorization.k8s.io
          name: system:azure-cloud-provider
        subjects:
          - kind: ServiceAccount
            name: persistent-volume-binder
            namespace: kube-system

        ```
    -   Create a PV
        ```yaml
        apiVersion: v1
        kind: PersistentVolume
        metadata:
          name: azurefile1
        spec:
          capacity:
            storage: 1Gi
          accessModes:
            - ReadWriteMany
          storageClassName: ""
          azureFile:
            secretName: azure-secret
            shareName: zdenek-vicar
            readOnly: false
          mountOptions:
          - dir_mode=0777
          - file_mode=0777
          - uid=1000
          - gid=1000
        ```
    -   Create a PVC
        ```yaml
        apiVersion: v1
        kind: PersistentVolumeClaim
        metadata:
          name: azurefile
        spec:
          accessModes:
            - ReadWriteMany
          storageClassName: ""
          resources:
            requests:
              storage: 1Gi
        ```
    -   Create a Pod with PVC used
        ```yaml
        apiVersion: v1
        kind: Pod
        metadata:
          name: volume-pvc
        spec:
          containers:
            - name: nginx-pvc
              image: nginx:1.17-alpine
              ports:
                - containerPort: 80
              volumeMounts:
                - name: azurefile
                  mountPath: /data/azurefile
        volumes:
          - name: azurefile
            persistentVolumeClaim:
              claimName: azurefile
        ```
### Documentation materials:
-   [Configure a Pod to Use a Volume for Storage](https://v1-16.docs.kubernetes.io//docs/tasks/configure-pod-container/configure-volume-storage/)
