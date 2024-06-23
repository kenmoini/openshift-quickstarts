# Node Configuration - MachineConfig, SystemD Services

By default OpenShift runs on Red Hat CoreOS which is meant to be a minimal operating system just to run containers and Kubernetes.

Due to this, some services are not enabled by default.  Below are some examples on using MachineConfigs to enable and configure various services.

Additional documentation can be found here: https://docs.openshift.com/container-platform/4.15/post_installation_configuration/machine-configuration-tasks.html#using-machineconfigs-to-change-machines

## Enable iSCSId & Multipathd

The following MachineConfig enables the iSCSId and Multipathd services on worker nodes.  For other nodes, make sure it matches the MachineConfigPool name intended.

```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-custom-enable-iscsid-worker
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.2.0
    systemd:
      units:
        - enabled: true
          name: iscsid.service
        - enabled: true
          name: multipathd.service
```

## Create a One-shot Service

This example will create a one-shot service that starts at boot.  Its goal is to set some SELinux contexts so that containers can use NFS services such as with the Nutanix Files CSI.

```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 98-worker-nfs-pv-selinux-workaround
spec:
  config:
    ignition:
      version: 3.2.0
    systemd:
      units:
        - contents: |
            [Unit]
            Description=NFS PV selinux error workaround
            [Service]
            Type=oneshot
            ExecStart=/bin/sh -c '/sbin/semanage permissive -l | /bin/grep -qw container_init_t || /sbin/semanage permissive -a container_init_t'
            [Install]
            WantedBy=multi-user.target
          enabled: true
          name: nfs-pv-selinux-workaround.service
```
