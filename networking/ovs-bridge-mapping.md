# OpenShift Networking - OVS Bridge Mapping

## Bridge to Physical Management Network

**User Story:** I installed my cluster in the 192.168.42.0/23 subnet, I want my Pods/VMs to also have access to this network via a bridge device.

When OpenShift is installed, the interface/bond/VLAN that it is installed to will be consumed by OVN-Kubernetes and added to the `br-ex` interface.  Thusly, you cannot create a bridge to that base interface that was used for the installation.  **Doing so will break your cluster!**

However, with the OVN-Kubernetes cluster network type, you can create a OVS Bridge Mapping to that interface that will expose Pods/VMs to the subnet that the nodes are on via that `br-ex` interface.

See additional documentation here: https://docs.openshift.com/container-platform/4.15/networking/multiple_networks/configuring-additional-network.html#configuring-additional-network_ovn-kubernetes-configuration-for-a-localnet-topology

### Label Nodes

Before you create NMState NetworkConfigurationPolicy CRs, you should label your nodes.  Of course you can target nodes by node labels such as hostname or role types, but it's advised to label nodes with the networks that they have access to.

For example, if you were creating a NodeNetworkConfigurationPolicy that was creating a VLAN 123 on a bond0, you could apply the node label `network.bond0-vlan123: enabled`.  This way you could have a multi-network, multi-tenant, multi-super-awesome-fun cluster that has some nodes available on some networks, and others available on others.

You can apply the labels via the Web UI, or via the CLI: `oc label node app-node-1 "network.baremetal-network=enabled"`

### Create a NodeNetworkConfigurationPolicy and NetworkAttachmentDefinition

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

---

## OVS Bridge Mappers

**User Story:** I have a NIC or a bond and access and/or trunked VLANs that I want to put Pods/VMs onto.

While you can use the combination of classic Linux VLANs/Bridges defined in NNCPs and NADs on those bridges, a more flexible and future-forward way is to use an OpenVSwitch Bridge defined in an NNCP, and VLANs defined in NADs.

The examples below will assume you have a Bond called `bond1`, and want to access the native access VLAN and trunked VLANs 100 and 200 on the network.

If you're coming from a vSphere sort of environment you can think of the following likenesses:

- NodeNetworkConfigurationPolicy - Physical Adapter configuration
- OVS Bridge - Distributed vSwitch
- NetworkAttachmentDefinition - Port Group

Follow the guidance above under **[Label Nodes](#label-nodes)** - an example Node Selector could be something like `in-band.bond.networks.example.io="enabled"`.  Of course start with 1 node, then increase the node selector scope to prevent any misconfigurations from impacting the cluster.

### Create an OVS Bridge and Mappers

The below OVS Bridge will be created on the `bond1` interface.  Note the number of entries in `.spec.desiredState.ovn.bridge-mappings` - there is one for the native/access/untagged VLAN, and one for each VLAN that will be available on this OVS Bridge.

```yaml
---
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ovs-br1
spec:
  nodeSelector:
    # Start with one host, then expand to a larger selector
    #kubernetes.io/hostname: worker-0
    #node-role.kubernetes.io/worker: ''
  desiredState:
    interfaces:
    - name: ovs-br1
      description: A dedicated OVS bridge with bond1 as a port allowing all VLANs and untagged traffic
      type: ovs-bridge
      state: up
      bridge:
        allow-extra-patch-ports: true
        options:
          # Turn to true if you need STP on your leaf nodes
          stp: false
        port:
        - name: bond1
    ovn:
      bridge-mappings:
      # Untagged/Access Mapper
      - localnet: ovs-br1
        bridge: ovs-br1
        state: present
      # Mapper per VLAN, different localnet
      - localnet: ovs-br100
        bridge: ovs-br1
        state: present
      - localnet: ovs-br200
        bridge: ovs-br1
        state: present
```

### Create NetworkAttachmentDefinitions for Networks on the OVS Bridge

Once that NNCP has been created, make sure it's been applied successfully and made available on the node(s) you have selected.  Following that, you can define the NADs for the different VLANs:

```yaml
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: ovs-br1
  # If the NAD is created in the default namespace it will be available cluster-wide.
  # If the NAD is made in another namespace, it will only be available for workloads there.
  namespace: default
spec:
  # cniVersion is required and must be "0.3.1"
  # type is required and must be "ovn-k8s-cni-overlay"
  # topology is probably localnet unless you want a network internal to the cluster only with layer2
  # netAttachDefName is required and must be "$NS/$NAME" of this NAD...yep.
  # name is required and is the name of the bridge-mapping.localnet!
  # VLAN is optional
  # MTU is optional
  # subnets is the subnets for the IP address space that will be MANAGED BY OPENSHIFT!  If you want to use DHCP, you can omit this.
  # excludeSubnets is the same thing as subnets, but in inverse with the same terms applied.
  config: |-
    {
      "cniVersion": "0.3.1",
      "type": "ovn-k8s-cni-overlay",
      "topology": "localnet",
      "netAttachDefName": "default/ovs-br1",
      "name": "ovs-br1",
      "vlanID": 100,
      "mtu": 1500
    }
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: ovs-br1-v100
  namespace: default
spec:
  config: |-
    {
      "cniVersion": "0.3.1",
      "type": "ovn-k8s-cni-overlay",
      "topology": "localnet",
      "netAttachDefName": "default/ovs-br1-v100",
      "name": "ovs-br100",
      "vlanID": 100,
      "mtu": 1500
    }
---
apiVersion: k8s.cni.cncf.io/v1
kind: NetworkAttachmentDefinition
metadata:
  name: ovs-br1-v200
  namespace: default
spec:
  config: |-
    {
      "cniVersion": "0.3.1",
      "type": "ovn-k8s-cni-overlay",
      "topology": "localnet",
      "netAttachDefName": "default/ovs-br1-v200",
      "name": "ovs-br200",
      "vlanID": 200,
      "mtu": 1500
    }
```

With the OVS Bridge and NADs created, you should be able to place a workload on those networks and route around them.
