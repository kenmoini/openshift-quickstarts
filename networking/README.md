# OpenShift Networking

The following examples in this folder are meant to get started with OpenShift Network, primarily used for OpenShfit Virtualize.  Mostly it answers the questions:

- [How do I access the subnet my OpenShift nodes are on?](ovs-bridge-mapping.md)
- [How do I define additional bonds, VLANs, and bridge devices?](./nmstate-examples.md)
- [How do I use defined bridges for VMs?](./nad-examples.md)
- [How do I define a Live Migration network for VMs?](./live-migration-network.md)
- [How do I use MetalLB to expose services that can't use layer 7 HTTP/HTTPS via the Ingress Router?](./metallb-deployment.md)

**Note**: The examples in this repository are meant for for POC/test environments - for production environments Red Hat Consulting Services are key since there are about a dozen different ways to configure additional networking interfaces for Pods and Virtual Machines and those services can guide you to the best option for your environment.

The following examples in this repository depend on using the [NMState Operator](./nmstate-operator.md), and most are tooled for OpenShift Virtualization.

## OpenShift Networking Primer

Regardless of what you do with OpenShift Networking it is crucial to understand the following:

- Whatever interface you deploy OpenShift to, be that a single NIC or a Bond, you cannot create a bridge to that interface.  The installation network is used by OVN-Kubernetes and creates an Open vSwitch interface called br-ex that consumes that interface.  This is considered the Management Network and again, **you cannot include that interface in a bridge - it will break the cluster!**
- To access the machine network/subnet that the cluster has been installed to, you must create a OVS bridge mapping.  See the example in [ovs-bridge-mapping.md](./ovs-bridge-mapping.md)
- NetworkAttachmentDefinitions (NADs) in the `default` Namespace are available cluster-wide.  NADs in other Namespaces are only available for Pods/VMs in those Namespaces.  This can provide cluster-wide networking as well as RBAC-controlled networks.