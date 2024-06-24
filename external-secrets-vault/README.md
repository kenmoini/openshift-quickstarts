# External Secrets Operator and Vault

A common practice is to leverage an external secrets platform and sync Secrets into OpenShift.  This is a better pattern for redistribution and access control.  This is a simple way to do so, there are other methods as well such as running a sidecar though that also introduces additional complexity and workload requirements.

Additional documentation can be found here: https://external-secrets.io/latest/provider/hashicorp-vault/

> User Story: I want to store secrets in Hashicorp Vault and have them synced into OpenShift when needed for a workload

## 1. Install the External Secrets Operator

```yaml=
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/external-secrets-operator.openshift-operators: ''
  name: external-secrets-operator
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Manual
  name: external-secrets-operator
  source: community-operators
  sourceNamespace: openshift-marketplace
```

Once the subscription is created you will need to approve the InstallPlan.

## 2. Create a Namespace to deploy the Operator Instance

```yaml=
---
apiVersion: v1
kind: Namespace
metadata:
  name: external-secrets-operator
```

## 3. Create a ConfigMap with any additional Trusted Root Certificate Authorities

This assumes that your additional trusted Root CAs are added to the cluster-wide configuration: [Custom Root CAs in OpenShift](https://kenmoini.com/post/2022/02/custom-root-ca-in-openshift/)

```yaml=
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: trusted-ca
  namespace: external-secrets-operator
  labels:
    config.openshift.io/inject-trusted-cabundle: 'true'
data: {}
```

## 4. Create the Operator Instance

The configuration below uses the UBI based ESO images so that you can mount the ConfigMap with the Root CAs to the system.  The image tag version must match the version of the installed Operator which is why the Subscription is set to a manual approval process.  Alternatively you can provide your Root CA that signs the Vault HTTPS certificate in the {Cluster}SecretStore spec.

```yaml=
---
kind: OperatorConfig
apiVersion: operator.external-secrets.io/v1alpha1
metadata:
  name: eso
  namespace: external-secrets-operator
spec:
  global:
    nodeSelector: {}
    #nodeSelector:
    #  node-role.kubernetes.io/infra: ""
    tolerations: []
    #tolerations:
    #  - effect: NoSchedule
    #    key: node-role.kubernetes.io/infra
    #    value: reserved
    #  - effect: NoExecute
    #    key: node-role.kubernetes.io/infra
    #    value: reserved
  affinity: {}
  extraVolumes:
    - configMap:
        items:
          - key: ca-bundle.crt
            path: tls-ca-bundle.pem
        name: trusted-ca
      name: trusted-ca
  extraVolumeMounts:
    - mountPath: /etc/pki/ca-trust/extracted/pem
      name: trusted-ca
      readOnly: true
  certController:
    affinity: {}
    create: true
    deploymentAnnotations: {}
    extraArgs: {}
    extraEnv: []
    fullnameOverride: ''
    image:
      pullPolicy: IfNotPresent
      repository: ghcr.io/external-secrets/external-secrets
      tag: 'v0.9.19-ubi'
    imagePullSecrets: []
    nameOverride: ''
    nodeSelector: {}
    #nodeSelector:
    #  node-role.kubernetes.io/infra: ""
    podAnnotations: {}
    podLabels: {}
    podSecurityContext: {}
    priorityClassName: ''
    prometheus:
      enabled: false
      service:
        port: 8080
    rbac:
      create: true
    requeueInterval: 5m
    resources: {}
    securityContext: {}
    serviceAccount:
      annotations: {}
      create: true
      name: ''
    tolerations: []
    #tolerations:
    #  - effect: NoSchedule
    #    key: node-role.kubernetes.io/infra
    #    value: reserved
    #  - effect: NoExecute
    #    key: node-role.kubernetes.io/infra
    #    value: reserved
  concurrent: 1
  controllerClass: ''
  crds:
    createClusterExternalSecret: true
    createClusterSecretStore: true
  createOperator: true
  deploymentAnnotations: {}
  extraArgs: {}
  extraEnv: []
  fullnameOverride: ''
  image:
    pullPolicy: IfNotPresent
    repository: ghcr.io/external-secrets/external-secrets
    tag: 'v0.9.19-ubi'
  imagePullSecrets: []
  installCRDs: false
  leaderElect: false
  nameOverride: ''
  nodeSelector: {}
  #nodeSelector:
  #  node-role.kubernetes.io/infra: ""
  podAnnotations: {}
  podLabels: {}
  podSecurityContext: {}
  priorityClassName: ''
  processClusterExternalSecret: true
  processClusterStore: true
  prometheus:
    enabled: false
    service:
      port: 8080
  rbac:
    create: true
  replicaCount: 1
  resources: {}
  scopedNamespace: ''
  scopedRBAC: false
  securityContext: {}
  serviceAccount:
    annotations: {}
    create: true
    name: ''
  tolerations: []
  #tolerations:
  #  - effect: NoSchedule
  #    key: node-role.kubernetes.io/infra
  #    value: reserved
  #  - effect: NoExecute
  #    key: node-role.kubernetes.io/infra
  #    value: reserved
  webhook:
    affinity: {}
    certCheckInterval: 5m
    certDir: /tmp/certs
    create: true
    deploymentAnnotations: {}
    extraArgs: {}
    extraEnv: []
    fullnameOverride: ''
    image:
      pullPolicy: IfNotPresent
      repository: ghcr.io/external-secrets/external-secrets
      tag: 'v0.9.19-ubi'
    imagePullSecrets: []
    nameOverride: ''
    nodeSelector: {}
    #nodeSelector:
    #  node-role.kubernetes.io/infra: ""
    podAnnotations: {}
    podLabels: {}
    podSecurityContext: {}
    priorityClassName: ''
    prometheus:
      enabled: false
      service:
        port: 8080
    rbac:
      create: true
    replicaCount: 1
    resources: {}
    securityContext: {}
    serviceAccount:
      annotations: {}
      create: true
      name: ''
    tolerations: []
    #tolerations:
    #  - effect: NoSchedule
    #    key: node-role.kubernetes.io/infra
    #    value: reserved
    #  - effect: NoExecute
    #    key: node-role.kubernetes.io/infra
    #    value: reserved
```

## 5. Create a Token in Hashicorp Vault

Log into Hashicorp Vault and create a Token that can be used to authenticate with.  This user will need read ACLs to the engines/paths that you want to sync.  Another option would be to leverage the Kubernetes authentication integration with Vault instead of using a Token.

## 6. Create the Secret in OpenShift with the Token

```yaml=
---
kind: Secret
apiVersion: v1
metadata:
  name: vault-token
  namespace: external-secrets-operator
stringData:
  token: hvs.someToken
type: Opaque
```

## 7. Create a {Cluster}SecretStore

A {Cluster}SecretStore defines the Vault endpoint that will be communicated with.  You can use a namespace-scoped SecretStore, or a cluster-wide ClusterSecretStore.

### ClusterSecretStore

```yaml=
---
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: kv-store
spec:
  provider:
    vault:
      auth:
        tokenSecretRef:
          key: token
          name: vault-token
          namespace: external-secrets-operator
      path: kv # path in vault to use as a base
      server: 'https://vault.apps.k8s.kemo.labs/'
      version: v2
```

### SecretStore

```yaml=
---
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: kv-store
  namespace: external-secrets-operator
spec:
  provider:
    vault:
      auth:
        tokenSecretRef:
          key: token
          name: vault-token
          namespace: external-secrets-operator
      path: kv
      server: 'https://vault.apps.k8s.kemo.labs/'
      version: v2
```

## 8. Create an ExternalSecret

### Opaque Secret Example

```yaml=
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: dummy-secret
  namespace: external-secrets-operator
spec:
  data:
    - remoteRef:
        conversionStrategy: Default
        decodingStrategy: None
        key: kemo-labs/credentials/dummy-secret # name of the secret under the target Vault path
        metadataPolicy: None
        property: superSecret # name of the key in that named Vault secret key
      secretKey: superSecret # the key to be made in this k8s Secret
  refreshInterval: 100s
  secretStoreRef:
    kind: ClusterSecretStore # SecretStore or ClusterSecretStore
    name: kv-store # name of the previously created SecretStore
  target:
    creationPolicy: Owner
    deletionPolicy: Retain
    name: dummy-secret # name of the k8s Secret to make in this namespace
    template:
      engineVersion: v2
      mergePolicy: Replace
      type: Opaque # type of k8s Secret to make
```

### Pull Secret Example

```yaml=
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: dummy-secret
  namespace: external-secrets-operator
spec:
  data:
    - remoteRef:
        conversionStrategy: Default
        decodingStrategy: None
        key: kemo-labs/credentials/pull-secrets # name of the secret under the target Vault path
        metadataPolicy: None
        property: privatePullSecret # name of the key in that named Vault secret key
      secretKey: .dockerconfigjson # the key to be made in this k8s Secret
  refreshInterval: 100s
  secretStoreRef:
    kind: ClusterSecretStore # SecretStore or ClusterSecretStore
    name: kv-store # name of the previously created SecretStore
  target:
    creationPolicy: Owner
    deletionPolicy: Retain
    name: jfrog-pull-secret # name of the k8s Secret to make in this namespace
    template:
      engineVersion: v2
      mergePolicy: Replace
      type: kubernetes.io/dockerconfigjson # type of k8s Secret to make
```

## 9.  Attach the Secret to a Workload

## 10. ???????????

## 11. PROFIT!!!!1