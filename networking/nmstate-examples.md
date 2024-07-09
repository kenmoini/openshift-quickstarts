# OpenShift Networking - NMState Examples

**User Story:** I need some examples of configuring additional bonds, VLANs, and bridge interfaces on my cluster hosts.

Once you have the [NMState Operator installed](./nmstate-operator.md), you can begin to configure additional network interfaces.  Below you can find some examples of how to create some common interfaces.

**Note:** Before applying any NMState configuration to multiple hosts, it's advised to apply it to a single host and confirm the operation of it.  Otherwise a misconfigured NNCP can break your cluster.

It's advised to target network specific Node labels instead of machine pools.

***WARNING:*** You cannot create a bridge device to the interface that was used during install as it is consumed by OVN-Kubernetes for the system managed `br-ex` interface!  You also cannot convert a single NIC used during install to a bond.  You may however configure additional VLANs on the base interface and bridges to that, eg:

- [OK] eth0 used during installation
- [BAD] Creating a bond0 post-install with eth0 and eth1
- [BAD] Creating bridgeNN on eth0 post-install

- [OK] bond0 with eth0 and eth1 used during installation
- [BAD] Creating bridgeNN on bond0 post-install

- [OK] bond0.99 VLAN on bond0 used during install
- [BAD] Creating bridgeNN on bond0.99 post-install
- [OK] Creating bond0.123 VLAN on bond0 post-install
- [OK] Creating bridgeNN on bond0.123 VLAN post-install

Additional documentation can be found here: https://docs.openshift.com/container-platform/4.15/networking/k8s_nmstate/k8s-nmstate-updating-node-network-config.html

## Bonded NICs

```yaml
---
apiVersion: nmstate.io/v1beta1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: bond0-ens5f2-ens5f3
spec:
  nodeSelector: # make sure to set a selector - start with 1 host, then apply to more
    #kubernetes.io/hostname: app-node-1.ocp.example.com
    #bond0-ens5f2-ens5f3: "true"
  desiredState:
    interfaces:
      - name: bond0
        type: bond
        state: up
        # ipv4 configuration is optional if this will be used by a bridge
        #ipv4:
        #  enabled: true
        #  dhcp: true
        #  address: # static IP configuration
        #    - ip: 192.168.10.123
        #      prefix-length: 24
        link-aggregation:
          # mode=1 active-backup
          # mode=2 balance-xor
          # mode=4 802.3ad
          # mode=5 balance-tlb
          # mode=6 balance-alb
          mode: 802.3ad # LACP
          options:
            miimon: '100'
          port: # must be the same name on all hosts
            - ens5f2
            - ens5f3
```

## VLAN on a NIC

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vlan99-eth1
spec:
  nodeSelector: # make sure to set a selector - start with 1 host, then apply to more
    #kubernetes.io/hostname: app-node-1.ocp.example.com
    #vlan99-eth1: "true"
  desiredState:
    interfaces:
      - name: eth1.99
        description: VLAN 99 using eth1
        type: vlan
        state: up
        vlan:
          base-iface: eth1
          id: 99
```

## VLAN on a Bond

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: vlan123-bond1
spec:
  nodeSelector: # make sure to set a selector - start with 1 host, then apply to more
    #kubernetes.io/hostname: app-node-1.ocp.example.com
    #vlan123-bond1: "true"
  desiredState:
    interfaces:
      - name: bond1.123
        description: VLAN 123 using bond1
        type: vlan
        state: up
        vlan:
          base-iface: bond1
          id: 123
```

## Bridge on a NIC

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-eth1
spec:
  nodeSelector: # make sure to set a selector - start with 1 host, then apply to more
    #kubernetes.io/hostname: app-node-1.ocp.example.com
    #br1-eth1: "true"
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge with eth1 as a port
        type: linux-bridge
        state: up
        # ipv4 configuration is optional if this will be used by Pods/VMs
        #ipv4:
        #  enabled: true
        #  dhcp: true
        #  address: # static IP configuration
        #    - ip: 192.168.10.123
        #      prefix-length: 24
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: eth1
```

## Bridge on a Bond

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-bond1
spec:
  nodeSelector: # make sure to set a selector - start with 1 host, then apply to more
    #kubernetes.io/hostname: app-node-1.ocp.example.com
    #br1-bond1: "true"
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge with bond1 as a port
        type: linux-bridge
        state: up
        # ipv4 configuration is optional if this will be used by Pods/VMs
        #ipv4:
        #  enabled: true
        #  dhcp: true
        #  address: # static IP configuration
        #    - ip: 192.168.10.123
        #      prefix-length: 24
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: bond1
```

## Bridge on a bonded VLAN

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: br1-bond1-vlan20
spec:
  nodeSelector: # make sure to set a selector - start with 1 host, then apply to more
    #kubernetes.io/hostname: app-node-1.ocp.example.com
    #br1-bond1-vlan20: "true"
  desiredState:
    interfaces:
      - name: br1
        description: Linux bridge with bond1 VLAN 20 as a port
        type: linux-bridge
        state: up
        # ipv4 configuration is optional if this will be used by Pods/VMs
        #ipv4:
        #  enabled: true
        #  dhcp: true
        #  address: # static IP configuration
        #    - ip: 192.168.10.123
        #      prefix-length: 24
        bridge:
          options:
            stp:
              enabled: false
          port:
            - name: bond1.20
```

Note that you can configure multiple interfaces in a single NNCP, however it's not advised as it is not as maintainable by mere mortals and introduces instances where networking issues can arise for multiple interfaces in case one is misconfigured.
