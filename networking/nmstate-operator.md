# Networking - NMState Operator

**User Story:** I need to define additional interfaces post-install.

In OpenShift the Red Hat CoreOS nodes have their configuration set via NetworkManager.  Instead of creating MachineConfigs or doing any other sort of manual editing, you can use the NMState Operator to set networking configuration for nodes.  This is a typical requirement for OpenShift Virtualization.

Installing the Operator is a pretty simple process, and once it is enabled you'll have access to new sections in the Administrator Web UI under Networking to set NodeNetworkConfigurationPolicy CRs, see the applied network interfaces via the NodeNetworkStates, and create NetworkAttachmentDefinitions that map to various interfaces to be consumed by Pods and VMs.

Additional documentation can be found here: https://docs.openshift.com/container-platform/4.15/networking/k8s_nmstate/k8s-nmstate-about-the-k8s-nmstate-operator.html

## Installing the NMState Operator

You can install the NMState Operator via the Web UI and the Operator Hub, or via the following manifest objects:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: openshift-nmstate
    name: openshift-nmstate
  name: openshift-nmstate
spec: {}
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  annotations:
    olm.providedAPIs: NMState.v1.nmstate.io
  name: openshift-nmstate
  namespace: openshift-nmstate
spec:
  targetNamespaces:
  - openshift-nmstate
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/kubernetes-nmstate-operator.openshift-nmstate: ""
  name: kubernetes-nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubernetes-nmstate-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

## Deploying the NMState Operator Instance

Once the operator is installed, you can deploy the NMState operator workloads by creating an NMState CR - the default should work for most instances:

```yaml
---
apiVersion: nmstate.io/v1
kind: NMState
metadata:
  name: nmstate
```

With the Operator and Instance deployed, you should see a notice in the Web UI about a new version of the console being available.  Refresh your browser to see the new links under the Networking section of the navigation pane.  Now you can continue to [define additional network interfaces](./nmstate-examples.md).

**Note:** It is a best practice not to target MachinePools/Roles when applying NodeNetworkConfigurationPolicy CRs.  Instead, create a specific label for whatever networking interfaces you are defined and apply them to the node, then have the NNCPs target those network specific labels.  eg if you are adding an additional bonded interface, bond1, then label your nodes with something like `networking.bond1: enabled`