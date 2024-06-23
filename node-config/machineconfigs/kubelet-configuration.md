# Node Configuration - Kubelet Configuration

The Kubelet configuration sets some important tuning specifications for the cluster and API.

Additional documentation can be found here: https://docs.openshift.com/container-platform/4.15/post_installation_configuration/machine-configuration-tasks.html#create-a-kubeletconfig-crd-to-edit-kubelet-parameters_post-install-machine-configuration-tasks

## Common Single Node OpenShift Configuration

On a large Single Node OpenShift instance you may need to run more than the default 250 Pod Limit set per node.  Below you can find a KubeletConfig to do so - note that for SNO instances the MachineConfigPool is set to `master`:

```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: set-max-pods-master
spec:
  machineConfigPoolSelector:
    matchLabels: # label on a MachineConfigPool to match - you can set custom labels on multiple MCPs if you want to target an ensemble
      pools.operator.machineconfiguration.openshift.io/master: ""
  kubeletConfig:
    podsPerCore: 0 # Disable pods per core allocation
    maxPods: 500 # Default 250, max 2500
    systemReserved: # Increase the reserved resources for non-kubernetes components
      cpu: 4000m
      memory: 8Gi
    kubeReserved: # Increase the reserved resources for kubernetes components
      cpu: 4000m
      memory: 8Gi
```