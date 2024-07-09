# OpenShift Networking - Dedicated Storage Networks for OpenShift Data Foundation/OpenShift Container Storage

When deploying OpenShift Data Foundation/OpenShift Container Storage, by default it will relay traffic over the cluster SDN.  You may optionally configure dedicated NICs to use for storage traffic.

There are 3 different types of traffic patterns:

- **Pod-to-pod traffic** - This is general network traffic that is relayed over the cluster SDN.
- **Pod-to-storage traffic** - Known as public network traffic, this can be configured to run over specific separate NICs
- **ODF internal traffic** - Known as data cluster network traffic, used by ODF for replication, rebalancing, etc

With Multus in OpenShift, you may configure Pod-to-storage and ODF internal traffic patterns to go over either a separate shared interface, or a set of two separate interfaces for additional segmentation.

Additional documentation can be found here: https://docs.redhat.com/en/documentation/red_hat_openshift_data_foundation/4.15/html/planning_your_deployment/network-requirements_rhodf

## NetworkAttachmentDefinition for shared Public Network and Data Cluster Network Traffic

If there is only one additional network interface available for storage traffic segmentation you can create a NAD that will separate both Pod-to-storage and ODF internal traffic onto that interface.

```yaml
# In-cluster IPAM
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ocs-public-cluster
  namespace: openshift-storage
spec:
  config: '{
  	"cniVersion": "0.3.1",
  	"type": "macvlan",
  	"master": "eth2",
  	"mode": "bridge",
  	"ipam": {
      "type": "whereabouts",
      "range": "192.168.1.0/24"
  	}
  }'
```

- The name and namespace above are not to be changed.
- The master/parent interface can be a basic ethernet NIC or bond.
- The master/parent interface must be the same on all storage and application nodes on the cluster, connected to the same underlying network.
- Defined interfaces need to be at least 10Gbit interfaces.
- The IPAM whereabouts plugin will manage leases in the specified subnet where a DHCP server is not available.  If a DHCP server is present on that subnet then IPAM can be set to DHCP as seen in the following example:

```yaml
# DHCP managed subnet
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ocs-public-cluster
  namespace: openshift-storage
spec:
  config: '{
  	"cniVersion": "0.3.1",
  	"type": "macvlan",
  	"master": "eth2",
  	"mode": "bridge",
  	"ipam": {
      "type": "dhcp"
  	}
  }'
```

## NetworkAttachmentDefinition for separate Public Network and Data Cluster Network Traffic

If there are two available NICs that can be dedicated for separate pod-to-storage network traffic and data cluster network traffic, then you may configure two separate NADs for additional segmentation:

```yaml
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ocs-public
  namespace: openshift-storage
spec:
  config: '{
  	"cniVersion": "0.3.1",
  	"type": "macvlan",
  	"master": "eth2",
  	"mode": "bridge",
  	"ipam": {
      "type": "whereabouts",
      "range": "192.168.1.0/24"
  	}
  }'
```

```yaml
---
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: ocs-cluster
  namespace: openshift-storage
spec:
  config: '{
  	"cniVersion": "0.3.1",
  	"type": "macvlan",
  	"master": "eth3",
  	"mode": "bridge",
  	"ipam": {
      "type": "whereabouts",
      "range": "192.168.2.0/24"
  	}
  }'
```

- The name and namespace above are not to be changed.
- The master/parent interface can be a basic ethernet NIC or bond.
- The master/parent interface for public/pod-to-storage must be the same on all storage and application nodes on the cluster, connected to the same underlying network.
- The master/parent interface for data cluster traffic must be the same on all storage nodes on the cluster, connected to the same underlying network.
- Defined interfaces need to be at least 10Gbit interfaces.