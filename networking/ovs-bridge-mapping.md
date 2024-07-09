# OpenShift Networking - OVS Bridge Mapping

**User Story:** I installed my cluster in the 192.168.42.0/23 subnet, I want my Pods/VMs to also have access to this network via a bridge device.

When OpenShift is installed, the interface/bond/VLAN that it is installed to will be consumed by OVN-Kubernetes and added to the `br-ex` interface.  Thusly, you cannot create a bridge to that base interface that was used for the installation.  **Doing so will break your cluster!**

However, with the OVN-Kubernetes cluster network type, you can create a OVS Bridge Mapping to that interface that will expose Pods/VMs to the subnet that the nodes are on via that `br-ex` interface.

See additional documentation here: https://docs.openshift.com/container-platform/4.15/networking/multiple_networks/configuring-additional-network.html#configuring-additional-network_ovn-kubernetes-configuration-for-a-localnet-topology

## Label Nodes

Before you create NMState NetworkConfigurationPolicy CRs, you should label your nodes.  Of course you can target nodes by node labels such as hostname or role types, but it's advised to label nodes with the networks that they have access to.

For example, if you were creating a NodeNetworkConfigurationPolicy that was creating a VLAN 123 on a bond0, you could apply the node label `network.bond0-vlan123: enabled`.  This way you could have a multi-network, multi-tenant, multi-super-awesome-fun cluster that has some nodes available on some networks, and others available on others.

You can apply the labels via the Web UI, or via the CLI: `oc label node app-node-1 "network.baremetal-network=enabled"`

## Create a NodeNetworkConfigurationPolicy and NetworkAttachmentDefinition

With the [NMState Operator installed](./nmstate-operator.md), you can create a NodeNetworkConfigurationPolicy that will define a OVN Bridge Mapping:

```yaml
---
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ovs-br-ex-bridge
spec:
  desiredState:
    ovn:
      bridge-mappings:
        - bridge: br-ex
          localnet: baremetal-network
          state: present
  nodeSelector:
    baremetal-network: enabled
```

With that NNCP in place, you can create a Network Attachment Definition that will consume that OVS bridge map which will make it available to Pods/VMs:

```yaml
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  annotations:
    description: baremetal-network connection for VMs
  name: baremetal-network
  namespace: default
spec:
  config: |-
    {
      "cniVersion": "0.3.1", 
      "name": "baremetal-network", 
      "type": "ovn-k8s-cni-overlay", 
      "topology": "localnet", 
      "promiscMode": true,
      "netAttachDefName": "default/baremetal-network",
      "ipam": {}
    }
```

Placing the NAD in the `default` Namespace will make it available cluster-wide.  If you place it in any other Namespace it will only be available to workloads in that specific Namespace.

Once the NNCP has been applied to the hosts and the NAD has been created, you can go about attaching the network to your workloads that will give it an IP on your host network in that 192.168.42.0/23 subnet mentioned at the top of this document.  If you're using a different subnet there's no other configuration required, that subnet was used for the User Story/example.