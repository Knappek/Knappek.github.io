# Deploy TKG packages in airgapped environments

## Prerequisites

- TKGS Supervisor cluster running
- embedded Harbor running
- shared services cluster running
- imgpkg installed

## Environment info

- vCenter 7u3p (Build 22837322)
- Supervisor cluster version v1.25.6+vmware.wcp.2
- Guest Cluster version: v1.23.8---vmware.3-tkg.1

## Procedure

### Deploy TKG Package Repository

### Install kapp-controller

We are following the official docs [here](https://docs.vmware.com/en/VMware-Tanzu-Packages/2024.4.12/tanzu-packages/kubectl-v7-kapp.html).

Run the following from a machine with access to the VMware public registry:

1. list available versions:

    ```
    imgpkg tag list -i projects.registry.vmware.com/tkg/kapp-controller
    ```

1. Copy the version of choice to your local registry

    ```shell
    imgpkg copy \
      -i projects.registry.vmware.com/tkg/kapp-controller:v0.30.0_vmware.1 \
      --to-repo 172.30.4.131/shared-services/kapp-controller \
      --registry-ca-cert-path ./ca.crt
    ```
    
    Alternatively, you can download the tar file to your filesystem

    ```shell
    imgpkg copy \
      -i projects.registry.vmware.com/tkg/kapp-controller:v0.41.7_vmware.1 \
      --to-tar  ./kapp-controller_v0.41.7_vmware.1
    ```

1. Create the `tanzu-package-repo-global` namespace: `kubectl create ns tanzu-package-repo-global`
1. create a secret to be able to pull images from the local registry with authentication

    ```shell
    kubectl create secret docker-registry embedded-harbor \
      --docker-server=172.30.4.131 \
      --docker-username=administrator@vsphere.local \
      --docker-password=VMware1! \
      -n tanzu-package-repo-global
    ```

1. Copy the content of the kapp-controller manifest from [here](https://docs.vmware.com/en/VMware-Tanzu-Packages/2024.4.12/tanzu-packages/kubectl-v7-kapp.html#manifest-for-kubernetes-v124-or-earlier-2) and make some changes:
   1. update the `image:` accordingly to point to your image stored in your local container registry
   1. add the following to `Deployment.spec.template.spec`

        ```shell
        imagePullSecrets:
        - name: embedded-harbor
        ```

1. Switch kubectl context to your shared services cluster and apply the manifest

    ```shell
    kubectl apply -f kapp-controller.yaml
    ```

### Add Package Repository to Cluster

We are following the official docs [here](https://docs.vmware.com/en/VMware-Tanzu-Packages/2024.4.12/tanzu-packages/prep.html#add-the-package-repository-to-the-cluster-2).

Run the following from a machine with access to the VMware public registry:

1. list available Package Repository versions:

    ```shell
    imgpkg tag list -i projects.registry.vmware.com/tkg/packages/standard/repo
    ```

1. Copy your version of choice to your registry

    ```shell
    imgpkg copy \
      -b projects.registry.vmware.com/tkg/packages/standard/repo:v1.6.1 \
        --to-repo 172.30.4.131/shared-services/packages/standard/repo:v1.6.1 \
        --registry-ca-cert-path ./ca.crt
    ```

1. Create a `PackageRepository` manifest and call it `packagerepo-v1.6.1.yaml`:

      ```yaml
      apiVersion: packaging.carvel.dev/v1alpha1
      kind: PackageRepository
      metadata:
        name: tanzu-standard
        namespace: tanzu-package-repo-global
      spec:
        fetch:
          imgpkgBundle:
            image: 172.30.4.131/shared-services/packages/standard/repo:v1.6.1
            secretRef:
              name: embedded-harbor
      ```

1. Switch kubectl context to your shared services cluster and apply the manifest

    ```shell
    kubectl apply -f packagerepo-v1.6.1.yaml
    ```

### Prepare user managed Tanzu Packages

1. Create a common namespace used for all user managed Tanzu packages: 

    ```shell
    kubectl create ns tanzu-packages-user-managed
    ```

1. Replicate the `embedded-harbor` secret from the `tanzu-package-repo-global` namespace to the `tanzu-packages-user-managed` namespace:

    ```shell
    kubectl create secret docker-registry embedded-harbor \
      --docker-server=172.30.4.131 \
      --docker-username=administrator@vsphere.local \
      --docker-password=VMware1! \
      -n tanzu-packages-user-managed
    ```

1. To use the `embedded-harbor` in all cert-manager deployment's `spec.template.spec.imagePullSecrets` we have to create a ytt overlay and use that overlay in the `PackageInstall`. Create the overlay `image-pull-secrets-overlay-deployment.yaml`

    ```yaml
    #@ load("@ytt:overlay", "overlay")
    #@overlay/match by=overlay.subset({"kind": "Deployment"}), expects="1+"
    ---
    spec:
      template:
        spec:
          #@overlay/match missing_ok=True
          imagePullSecrets:
          - name: embedded-harbor
    ```

    and then create a Kubernetes secret:

    ```shell
    kubectl create secret generic image-pull-secret-overlay-deployment \
      --from-file=image-pull-secrets-overlay-deployment.yaml \
      -n tanzu-packages-user-managed
    ```

1. We do the same for DaemonSets. Create the overlay `image-pull-secrets-overlay-daemonset.yaml`

    ```yaml
    #@ load("@ytt:overlay", "overlay")
    #@overlay/match by=overlay.subset({"kind": "DaemonSet"}), expects="1+"
    ---
    spec:
      template:
        spec:
          #@overlay/match missing_ok=True
          imagePullSecrets:
          - name: embedded-harbor
    ```

    and then create a Kubernetes secret:

    ```shell
    kubectl create secret generic image-pull-secret-overlay-daemonset \
      --from-file=image-pull-secrets-overlay-daemonset.yaml \
      -n tanzu-packages-user-managed
    ```

1. We do the same for StatefulSets. Create the overlay `image-pull-secrets-overlay-statefulsets.yaml`

    ```yaml
    #@ load("@ytt:overlay", "overlay")
    #@overlay/match by=overlay.subset({"kind": "StatefulSet"}), expects="1+"
    ---
    spec:
      template:
        spec:
          #@overlay/match missing_ok=True
          imagePullSecrets:
          - name: embedded-harbor
    ```

    and then create a Kubernetes secret:

    ```shell
    kubectl create secret generic image-pull-secret-overlay-statefulsets \
      --from-file=image-pull-secrets-overlay-statefulsets.yaml \
      -n tanzu-packages-user-managed
    ```

### Install cert-manager

We are following the official docs [here](https://docs.vmware.com/en/VMware-Tanzu-Packages/2024.4.12/tanzu-packages/packages-cert-mgr-super.html#kubectl).

1. Create the `cert-manager` namespace:
  
    ```shell
    kubectl create ns cert-manager
    ```

1. Create the `embedded-harbor` secrets used in `imagePullSecrets` in each pod:

    ```shell
    kubectl create secret docker-registry embedded-harbor \
      --docker-server=172.30.4.131 \
      --docker-username=administrator@vsphere.local \
      --docker-password=VMware1! \
      -n cert-manager
    ```

1. List available version for cert-manager:
  
    ```shell
    kubectl get packages -n tanzu-packages-user-managed
    ```

1. Create the manifest `cert-manager.yaml`:

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: cert-manager-sa
      namespace: tanzu-packages-user-managed
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: ServiceAccount
        name: cert-manager-sa
        namespace: tanzu-packages-user-managed
    ---
    apiVersion: packaging.carvel.dev/v1alpha1
    kind: PackageInstall
    metadata:
      name: cert-manager
      namespace: tanzu-packages-user-managed
      annotations:
        ext.packaging.carvel.dev/fetch-0-secret-name: embedded-harbor
        ext.packaging.carvel.dev/ytt-paths-from-secret-name.0: image-pull-secret-overlay-deployment
    spec:
      serviceAccountName: cert-manager-sa
      packageRef:
        refName: cert-manager.tanzu.vmware.com
        versionSelection:
          constraints: 1.7.2+vmware.1-tkg.1
      values:
      - secretRef:
          name: cert-manager-data-values
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: cert-manager-data-values
      namespace: tanzu-packages-user-managed
    stringData:
      values.yml: |
        ---
        namespace: cert-manager
    ```

    and apply it to the cluster: `kubectl apply -f cert-manager.yaml`

### Install Contour

The process is very simlar to installing `cert-manager`. We are following the official docs [here](https://docs.vmware.com/en/VMware-Tanzu-Packages/2024.4.12/tanzu-packages/packages-contour-super.html) using `kubectl`.

1. Create the `tanzu-system-ingress` namespace:
  
    ```shell
    kubectl create ns tanzu-system-ingress
    ```

1. Create the `embedded-harbor` secrets used in `imagePullSecrets` in each pod:

    ```shell
    kubectl create secret docker-registry embedded-harbor \
      --docker-server=172.30.4.131 \
      --docker-username=administrator@vsphere.local \
      --docker-password=VMware1! \
      -n tanzu-system-ingress
    ```

1. List available version for contour:
  
    ```shell
    kubectl get packages -n tanzu-packages-user-managed
    ```

1. Create the manifest `contour.yaml`

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: contour-sa
      namespace: tanzu-packages-user-managed
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: contour-role-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: ServiceAccount
        name: contour-sa
        namespace: tanzu-packages-user-managed
    ---
    apiVersion: packaging.carvel.dev/v1alpha1
    kind: PackageInstall
    metadata:
      name: contour
      namespace: tanzu-packages-user-managed
      annotations:
        ext.packaging.carvel.dev/fetch-0-secret-name: embedded-harbor
        ext.packaging.carvel.dev/ytt-paths-from-secret-name.0: image-pull-secret-overlay-deployment
        ext.packaging.carvel.dev/ytt-paths-from-secret-name.1: image-pull-secret-overlay-daemonset
    spec:
      serviceAccountName: contour-sa
      packageRef:
        refName: contour.tanzu.vmware.com
        versionSelection:
          constraints: 1.20.2+vmware.2-tkg.1
      values:
      - secretRef:
          name: contour-data-values
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: contour-data-values
      namespace: tanzu-packages-user-managed
    stringData:
      values.yml: |
        ---
        infrastructure_provider: vsphere
        namespace: tanzu-system-ingress
        contour:
          configFileContents: {}
          useProxyProtocol: false
          replicas: 2
          pspNames: "vmware-system-restricted"
          logLevel: info
        envoy:
          service:
            type: LoadBalancer
            annotations: {}
            nodePorts:
              http: null
              https: null
            externalTrafficPolicy: Cluster
            disableWait: false
          hostPorts:
            enable: false
            http: 80
            https: 443
          hostNetwork: false
          terminationGracePeriodSeconds: 300
          logLevel: info
          pspNames: null
        certificates:
          duration: 8760h
          renewBefore: 360h
    ```

    and apply it to the cluster: `kubectl apply -f contour.yaml`

### Install Harbor

1. Create the `tanzu-system-registry` namespace:
  
    ```shell
    kubectl create ns tanzu-system-registry
    ```

1. Create the `embedded-harbor` secrets used in `imagePullSecrets` in each pod:

    ```shell
    kubectl create secret docker-registry embedded-harbor \
      --docker-server=172.30.4.131 \
      --docker-username=administrator@vsphere.local \
      --docker-password=VMware1! \
      -n tanzu-system-registry
    ```

1. List available version for harbor:
  
    ```shell
    kubectl get packages -n tanzu-packages-user-managed
    ```

1. Create the manifest `harbor.yaml`:

    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: harbor-sa
      namespace: tanzu-packages-user-managed
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: habor-role-binding
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
      - kind: ServiceAccount
        name: harbor-sa
        namespace: tanzu-packages-user-managed
    ---
    apiVersion: packaging.carvel.dev/v1alpha1
    kind: PackageInstall
    metadata:
      name: harbor
      namespace: tanzu-packages-user-managed
      annotations:
        ext.packaging.carvel.dev/fetch-0-secret-name: embedded-harbor
        ext.packaging.carvel.dev/ytt-paths-from-secret-name.0: image-pull-secret-overlay-deployment
        ext.packaging.carvel.dev/ytt-paths-from-secret-name.1: image-pull-secret-overlay-statefulsets
    spec:
      serviceAccountName: harbor-sa
      packageRef:
        refName: harbor.tanzu.vmware.com
        versionSelection:
          constraints: 2.3.3+vmware.1-tkg.1
      values:
      - secretRef:
          name: harbor-data-values
    ---
    apiVersion: v1
    kind: Secret
    metadata:
      name: harbor-data-values
      namespace: tanzu-packages-user-managed
    stringData:
      values.yml: |
        namespace: tanzu-system-registry
        hostname: harbor.internal
        port:
          https: 443
        logLevel: info
        tlsCertificate:
          tls.crt: ""
          tls.key: ""
          ca.crt:
        tlsCertificateSecretName:
        enableContourHttpProxy: true
        harborAdminPassword: 'VMware1!'
        secretKey: 'aiGhooghu8uaS7zo'
        database:
          password: 'VMware1!'
          shmSizeLimit:
          maxIdleConns:
          maxOpenConns:
        exporter:
          cacheDuration:
        core:
          replicas: 1
          secret: 'VMware1!'
          xsrfKey: oopoo7iecae8wai5eejeethaingeip4W
        jobservice:
          replicas: 1
          secret: 'VMware1!'
        registry:
          replicas: 1
          secret: 'VMware1!'
        notary:
          enabled: true
        trivy:
          enabled: true
          replicas: 1
          gitHubToken: ""
          skipUpdate: false
        persistence:
          persistentVolumeClaim:
            registry:
              existingClaim: ""
              storageClass: "tkgs-storage-policy"
              subPath: ""
              accessMode: ReadWriteOnce
              size: 50Gi
            jobservice:
              existingClaim: ""
              storageClass: "tkgs-storage-policy"
              subPath: ""
              accessMode: ReadWriteOnce
              size: 10Gi
            database:
              existingClaim: ""
              storageClass: "tkgs-storage-policy"
              subPath: ""
              accessMode: ReadWriteOnce
              size: 10Gi
            redis:
              existingClaim: ""
              storageClass: "tkgs-storage-policy"
              subPath: ""
              accessMode: ReadWriteOnce
              size: 10Gi
            trivy:
              existingClaim: ""
              storageClass: "tkgs-storage-policy"
              subPath: ""
              accessMode: ReadWriteOnce
              size: 10Gi
        proxy:
          httpProxy:
          httpsProxy:
          noProxy: 127.0.0.1,localhost,.local,.internal
        pspNames: vmware-system-restricted,vmware-system-privileged
        network:
          ipFamilies: ["IPv4", "IPv6"]
    ```

    and apply it to the cluster: `kubectl apply -f harbor.yaml`
