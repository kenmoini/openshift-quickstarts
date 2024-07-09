# OpenShift Networking - Live Migration Network

**User Story:** I want to configure a specific Live Migration network for my VMs in my 192.168.99.0/24 network.

In order to define an additional network dedicated to live migrations, you need a separate NIC from your installed/management network and general VM networks - otherwise there's not much of a point really.  A common configuration found would be bond0 used for installation/management via 1GbE NICs, bond1 for VM traffic via 10G/25G/etc NICs, and another set of 10G/25G/etc NICs on bond2 for live migration traffic.  A separate NIC/bond for Live Migration traffic isn't a hard requirement as much as it is a best practice - you could use a VLAN on the management/VM network if that's the only option or eg where you have QoS set up for specific VLANs.

Another requirement, is a subnet that is separate from both management and VM traffic networks that is available on all hosts.

Additional documentation can be found here: https://docs.openshift.com/container-platform/4.15/virt/vm_networking/virt-dedicated-network-live-migration.html#virt-dedicated-network-live-migration

## Creating the NetworkAttachmentDefinition

Assuming your base network interface is already configured, the following NAD will configure a live migration network on bond2 - the bond2 interface can be switched out for any other network interface.

```yaml
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: migration-bridge
  namespace: openshift-cnv
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "migration-bridge",
    "type": "macvlan",
    "master": "bond2",
    "mode": "bridge",
    "ipam": {
      "type": "whereabouts", 
      "range": "192.168.99.0/24" 
    }
  }'
```

Note that it is required that the live migration NAD be provisioned in the `openshift-cnv` Namespace.  Adjust the `master` specification to target any other NIC, and change the IP range to match a routable network in your environment that is available in your environment across all virtualization hosts.

Also note that the mode is set to `bridge` however it does not need a bridge device configured prior to defining the NAD.

Once the NAD is created, you can configure the KubeVirt/OpenShift Virtualization operator to use it as a live migration network via the HyperConverged CR:

```yaml
apiVersion: hco.kubevirt.io/v1beta1
kind: HyperConverged
metadata:
  name: kubevirt-hyperconverged
spec:
  liveMigrationConfig:
    completionTimeoutPerGiB: 800
    network: migration-bridge
    parallelMigrationsPerCluster: 5
    parallelOutboundMigrationsPerNode: 2
    progressTimeout: 150
  # other config here...
```

You may also configure the Live Migration network by going into the Web UI, navigating to **Virtualization > Overview**, clicking on the **Settings** tab, then **Live Migration**.  The network should show in the **Live Migration Network** list.

Test this live migration by placing a Node into maintenance mode and checking a VirtualMachineInstance's status: `oc get vmi <vmi_name> -o jsonpath='{.status.migrationState.targetNodeAddress}'`
