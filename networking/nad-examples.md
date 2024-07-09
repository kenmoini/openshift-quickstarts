# OpenShift Networking - NetworkAttachmentDefinition Examples

To use interfaces defined by the NMState Operator with OpenShift Virtualization you need to create NetworkAttachmentDefinitions.  Below you can find some examples of common configuration.

## NAD to use a Bridge

The following example assumes you already have a Linux Bridge define via NMState's NNCP.

```yaml
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  annotations:
    k8s.v1.cni.cncf.io/resourceName: bridge.network.kubevirt.io/br-70
  name: br-70
  # Setting the namespace to default means this is cluster-scoped
  namespace: default
spec:
  config: |-
    {
      "name": "br-70",
      "type": "cnv-bridge",
      "cniVersion": "0.3.1",
      "bridge": "br-70",
      "macspoofchk": true,
      "ipam": {},
      "preserveDefaultVlan": false
    }
```

Replace the instances of `br-70` with the name of your [Linux Bridge defined in your NNCP](./nmstate-examples.md#bridge-on-a-bonded-vlan).