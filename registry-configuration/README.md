# Cluster Registry Configuration

Within OpenShift you can configure the platform to leverage private registries.  This is useful in cases where a private registry would have mirrored or proxied external registries.  You may also apply additional policies on allowed, disallowed, and insecure registries that the cluster may or may not pull images from.

Additional documentation can be found here: https://docs.openshift.com/container-platform/4.15/openshift_images/image-configuration.html

## Disconnected/Private Image Registries

If you are deploying OpenShift via a disconnected image registry that either have had images mirrored to it, or is acting as a pull-through/proxy cache, you can configure OpenShift to leverage that endpoint instead of public ones.  This can be configured during installation, or post-install.

> Before configuring any additional/disconnected registries, ensure credentials to access the registry have been added to the cluster-wide Pull Secret.  This can easily be appended by editing the Secret via the Web UI.

Create a set of ImageDigestSourcePolicy and ImageTagSourcePolicy CRs:

> Before applying these configurations, it's advised to perform a deployment/pull test via the mirror to ensure the cluster can authenticate and access the mirrors.  If access or transport fails, the cluster will enter a degraded state.

```yaml
---
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: private-mirror
spec:
  imageDigestMirrors:
    - mirrorSourcePolicy: NeverContactSource # Prevents continued attempts to pull the image from the source repository.
      source: quay.io
      mirrors:
        - quay-remote.registry.example.com

    - mirrorSourcePolicy: NeverContactSource
      source: registry.redhat.io
      mirrors:
        - registry-redhat-remote.registry.example.com

    - mirrorSourcePolicy: NeverContactSource
      source: registry.access.redhat.com
      mirrors:
        - registry-access-redhat-remote.registry.example.com

    - mirrorSourcePolicy: NeverContactSource
      source: registry.connect.redhat.com
      mirrors:
        - registry-connect-redhat-remote.registry.example.com
---
apiVersion: config.openshift.io/v1
kind: ImageTagMirrorSet
metadata:
  name: private-mirror
spec:
  imageTagMirrors:
    - mirrorSourcePolicy: NeverContactSource # Prevents continued attempts to pull the image from the source repository.
      source: quay.io
      mirrors:
        - quay-remote.registry.example.com

    - mirrorSourcePolicy: NeverContactSource
      source: registry.redhat.io
      mirrors:
        - registry-redhat-remote.registry.example.com

    - mirrorSourcePolicy: NeverContactSource
      source: registry.access.redhat.com
      mirrors:
        - registry-access-redhat-remote.registry.example.com

    - mirrorSourcePolicy: NeverContactSource
      source: registry.connect.redhat.com
      mirrors:
        - registry-connect-redhat-remote.registry.example.com
```

These two configuration files will tell OpenShift to pull images from the registry mirrors instead of contacting the public endpoints on the internet.

In some private image registry deployments, you may have a subpath for each mirrored/proxied registry endpoint instead of sub-domains.  Adjust as needed to access the target registries.

## Image Registry Policies

By default, any image registry is allowed so long as the cluster can authenticate to it via either the cluster-wide Pull Secret defined in the `pull-secret` Secret in the `openshift-config` Namespace, or if provided by individual deployments as an `imagePullSecret`.

In the case you want to specifically only allow certain registries, explicitly deny some, or allow non-HTTPs insecure connections to registries, you can configure the cluster to set those policies as well:

```yaml
---
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  name: cluster
spec:
  registrySources:
    allowedRegistries: # List of allowed registries
      - 'image-registry.openshift-image-registry.svc:5000'
      - virtual-proxy.jfrog-artifactory.d70.lab.kemo.network
      - jfrog-artifactory.d70.lab.kemo.network
      - disconn-harbor.d70.kemo.labs
      - ghcr.io
      - quay.io
      - registry.redhat.io
      - registry.access.redhat.com
      - registry.connect.redhat.com
    insecureRegistries: # List of registries to not validate TLS
      - virtual-proxy.jfrog-artifactory.d70.lab.kemo.network
      - jfrog-artifactory.d70.lab.kemo.network
    blockedRegistries: # registries/images to block entirely
      - docker.io
      - untrusted.com
      - reg1.io/myrepo/myapp:latest
```

## Custom Root CAs for Image Registries

If any registry has an HTTPS certificate that is signed by a non-standard Root CA, then you can provide the Root CA with the following configuration:

```yaml
# Create the ConfigMap storing the Roots
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: image-additional-trust-bundle
  namespace: openshift-config # must be in this namespace
  labels:
    config.openshift.io/inject-trusted-cabundle: 'true' # magic annotation to also create a `ca-bundle.crt` data key with the cluster-wide trusted bundle
data:
  # updateservice-registry is used by the cluster update graph service
  updateservice-registry: |
    -----BEGIN CERTIFICATE-----
    MIIGqzCCBJOgAwIBAgIUKMZCYZxHomZOUFLz8j0/ItBY/3cwDQYJKoZIhvcNAQEL
    ...
    zYML3t12ZU8JGpxxfUk2ObjKbixfSwSmTcWb+s8kgg==
    -----END CERTIFICATE-----
  # custom registries
  registry.example.com: |
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
  registry-with-port.example.com..5000: | 
    -----BEGIN CERTIFICATE-----
    ...
    -----END CERTIFICATE-----
```

```yaml
---
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  name: cluster
spec:
  # ...
  additionalTrustedCA:
    name: 'image-additional-trust-bundle'
  # ...
```