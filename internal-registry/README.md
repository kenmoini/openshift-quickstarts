# OpenShift Internal Image Registry

OpenShift provides an internal Image Registry service.  This is required when leveraging things such as OpenShift Builds, ImageStreams, or OpenShift AI.

By default the internal image registry is disabled due to it requiring a storage endpoint - this could be either PVC backed or an Object Store such as S3.  Below you'll find configuration for both of those deployment types, as well as additional configuration to expose the internal registry via a Route, moving the image registry to Infrastructure nodes, and how to configure additional trusted root certificates.

Additional documentation can be found here: https://docs.openshift.com/container-platform/4.15/registry/configuring-registry-operator.html

## PVC Backed Image Registry

To use cluster-local PVC storage for the internal registry it's important to note:

- A filesystem volume is needed.
- If more than 1 replica is deployed, the PVC must be ReadWriteMany.
- If ReadWriteOnce is only available then only 1 replica may be configured.

```yaml
---
# Create a PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry
  namespace: openshift-image-registry
spec:
  accessModes:
    - ReadWriteOnce # Ideally this would be ReadWriteMany, required for multiple replicas of the internal registry service
  resources:
    requests:
      storage: 100Gi # At least 100Gi is needed
  volumeMode: Filesystem # Must be Filesystem type
---
# Configure the internal image registry
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
spec:
  managementState: Managed # By default this is set to Unmanaged which disables the internal registry
  replicas: 1
  storage:
    managementState: Unmanaged # If this is set to Managed then the PVC will be automatically created with the default StorageClass
    pvc:
      claim: image-registry # name of the PVC previously created
  defaultRoute: true # Expose the image registry via a Route
  proxy: {} # any additional proxy configuration information, used by ImageStreams
  #  http: http://proxy.example.com:3128
  #  https: http://proxy.example.com:3128
  #  noProxy: .example.net
  # No need to really edit these things below here
  logLevel: Normal
  rolloutStrategy: Recreate
  operatorLogLevel: Normal
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  observedConfig: null
  unsupportedConfigOverrides: null
```

## S3-backed Internal Image Registry

An alternative to PVC-backed storage is an Object Store, eg S3-compatible buckets.  This allows the operation of multiple replicas.  Below is an example for configuring the internal image registry with OpenShift Data Foundations (ODF) - if using a different S3 service then you can simply skip the ObjectBucketClaim and configure the Secret and endpoint configuration to point to your alternative S3 service.

You can also find a guide to deploy Minio for testing purposes in this repo at [backup/oadp/deploy-minio.md](../backup/oadp/deploy-minio.md)

```yaml
---
# Create an ODF ObjectBucketClaim
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: internal-registry-bucket
  namespace: openshift-image-registry
  annotations:
    argocd.argoproj.io/sync-wave: "2"
  labels:
    app: noobaa
    bucket-provisioner: openshift-storage.noobaa.io-obc
    noobaa-domain: openshift-storage.noobaa.io
spec:
  additionalConfig:
    bucketclass: noobaa-default-bucket-class
  objectBucketName: obc-openshift-image-registry-internal-registry
  bucketName: internal-registry-bucket
  storageClassName: openshift-storage.noobaa.io
```

Note that the Secret created by ODF's OBC will not be in the format needed by the internal image registry.  In cases where you're deploying this via GitOps mechanisms, you can use the following RBAC and Job to sync the Secret to the proper format:

```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: noobaa-secret-sync
  namespace: openshift-image-registry
  annotations:
    argocd.argoproj.io/sync-wave: "2"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: noobaa-secret-sync
  namespace: openshift-image-registry
  annotations:
    argocd.argoproj.io/sync-wave: "2"
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - get
      - list
      - watch
      - create
      - update
      - patch
  - apiGroups:
      - ""
    resources:
      - pods
    verbs:
      - get
      - list
      - exec
  - apiGroups:
      - ""
    resources:
      - pods/exec
    verbs:
      - create
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: noobaa-secret-sync
  namespace: openshift-image-registry
  annotations:
    argocd.argoproj.io/sync-wave: "2"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: noobaa-secret-sync
subjects:
  - kind: ServiceAccount
    name: noobaa-secret-sync
    namespace: openshift-image-registry
---
apiVersion: batch/v1
kind: Job
metadata:
  name: noobaa-secret-sync
  namespace: openshift-image-registry
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  template:
    spec:
      containers:
      - name: noobaa-secret-sync
        image: registry.redhat.io/openshift4/ose-cli:latest
        imagePullPolicy: IfNotPresent
        command:
          - /bin/bash
          - -c
          - |
            #!/usr/bin/env bash

            until oc get secret -n openshift-image-registry internal-registry-bucket; do
                echo "Waiting for internal-registry-bucket secret to be created"
                sleep 5
            done

            KEY_ID=$(oc get secret -n openshift-image-registry internal-registry-bucket --template={{.data.AWS_ACCESS_KEY_ID}} | base64 -d)
            KEY_SECRET=$(oc get secret -n openshift-image-registry internal-registry-bucket --template={{.data.AWS_SECRET_ACCESS_KEY}} | base64 -d)

            cat <<EOF | oc apply -f -
            apiVersion: v1
            kind: Secret
            metadata:
              name: image-registry-private-configuration-user
              namespace: openshift-image-registry
            type: Opaque
            stringData:
              REGISTRY_STORAGE_S3_ACCESSKEY: ${KEY_ID}
              REGISTRY_STORAGE_S3_SECRETKEY: ${KEY_SECRET}
            EOF
      serviceAccountName: noobaa-secret-sync
      restartPolicy: OnFailure
```

Otherwise, you can configure the Secret for other S3 services in the following format:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: image-registry-private-configuration-user # must be name used to override default credential pointer
  namespace: openshift-image-registry
type: Opaque
stringData:
  REGISTRY_STORAGE_S3_ACCESSKEY: KEY_ID_HERE
  REGISTRY_STORAGE_S3_SECRETKEY: KEY_SECRET_HERE
```

With the bucket created and Secret in the proper format, you may proceed to configure the internal image registry:

```yaml
---
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
  annotations:
    argocd.argoproj.io/sync-wave: "5"
spec:
  managementState: Managed # By default this is set to Unmanaged which disables the internal registry
  replicas: 2 # Can be increased when S3 is configured
  storage:
    managementState: Unmanaged # Set this to Unmanaged when using non-integrated S3 endpoints
    s3:
      bucket: internal-registry-bucket # bucket name
      region: the-moon # Unless the S3 service has regions such as in AWS, set this to whatever
      regionEndpoint: https://s3-openshift-storage.apps.raza-gpu-sno.kemo.labs # Endpoint for 
      trustedCA:
        name: image-registry-s3-bundle # name of the ConfigMap in the openshift-config namespace with the trusted Root CA for the S3 service
  defaultRoute: true # Expose the image registry via a Route
  proxy: {} # any additional proxy configuration information, used by ImageStreams and to talk to S3
  #  http: http://proxy.example.com:3128
  #  https: http://proxy.example.com:3128
  #  noProxy: .example.net
  # No need to really edit these things below here
  logLevel: Normal
  rolloutStrategy: Recreate
  operatorLogLevel: Normal
  requests:
    read:
      maxWaitInQueue: 0s
    write:
      maxWaitInQueue: 0s
  observedConfig: null
  unsupportedConfigOverrides: null
```

If using an S3 service running in the OpenShift cluster such as ODF or Minio that is exposed via a Route, you can have OpenShift automatically create a trusted root CA bundle for you so long as the Root CA for the Route Ingress is configured in the cluster-wide additional trust bundle defined in the `user-ca-bundle` ConfigMap in the `openshift-config` namespace with the following annotated ConfigMap:

```yaml
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: image-registry-s3-bundle
  namespace: openshift-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"
  labels:
    config.openshift.io/inject-trusted-cabundle: 'true' # magic annotation
```

If it is not part of the cluster-wide additional trust bundle, provide the Root CA via that named ConfigMap with a data key of `ca-bundle.crt`.

## Running the Internal Registry on Infrastructure Nodes

OpenShift subscriptions only count against Application nodes.  In production environments it's a best practice to have Infrastructure nodes that can run services that support the cluster, such as this internal image registry.  Add the following configuration to the ImageRegistry Config to move the pods to tainted Infrastructure nodes:

```yaml
---
apiVersion: imageregistry.operator.openshift.io/v1
kind: Config
metadata:
  name: cluster
spec:
  # ...
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
    tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/infra
      value: reserved
    - effect: NoExecute
      key: node-role.kubernetes.io/infra
      value: reserved
  # ...
```