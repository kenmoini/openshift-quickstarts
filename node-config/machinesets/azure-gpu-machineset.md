## MachineSets - Creating a GPU MachineSet - Azure

If you're operating on-premise or with another infrastructure provider, skip this section.  Otherwise if you are running OpenShift in Azure you can run the following commands to copy down the default worker MachineSet and make some alterations to create a new MachineSet for adding GPU nodes to the cluster.

Find your desired Instance type here: https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview?tabs=breakdownseries%2Cgeneralsizelist%2Ccomputesizelist%2Cmemorysizelist%2Cstoragesizelist%2Cgpusizelist%2Cfpgasizelist%2Chpcsizelist#gpu-accelerated

The following instructions assume a multi-zone Azure deployment, eg in eastus1, eastus2, and eastus3.

```bash
# Get the Worker MachineSets and save them to a file
oc get machineset -n openshift-machine-api $(oc get machineset -n openshift-machine-api --selector machine.openshift.io/cluster-api-machine-type=worker -o jsonpath='{.items[0].metadata.name}') -o yaml > worker-$(oc get machineset -n openshift-machine-api --selector machine.openshift.io/cluster-api-machine-type=worker -o jsonpath='{.items[0].spec.template.spec.providerSpec.value.location}')-$(oc get machineset -n openshift-machine-api --selector machine.openshift.io/cluster-api-machine-type=worker -o jsonpath='{.items[0].spec.template.spec.providerSpec.value.zone}')-machineset.yml

oc get machineset -n openshift-machine-api $(oc get machineset -n openshift-machine-api --selector machine.openshift.io/cluster-api-machine-type=worker -o jsonpath='{.items[1].metadata.name}') -o yaml > worker-$(oc get machineset -n openshift-machine-api --selector machine.openshift.io/cluster-api-machine-type=worker -o jsonpath='{.items[1].spec.template.spec.providerSpec.value.location}')-$(oc get machineset -n openshift-machine-api --selector machine.openshift.io/cluster-api-machine-type=worker -o jsonpath='{.items[1].spec.template.spec.providerSpec.value.zone}')-machineset.yml

oc get machineset -n openshift-machine-api $(oc get machineset -n openshift-machine-api --selector machine.openshift.io/cluster-api-machine-type=worker -o jsonpath='{.items[2].metadata.name}') -o yaml > worker-$(oc get machineset -n openshift-machine-api --selector machine.openshift.io/cluster-api-machine-type=worker -o jsonpath='{.items[2].spec.template.spec.providerSpec.value.location}')-$(oc get machineset -n openshift-machine-api --selector machine.openshift.io/cluster-api-machine-type=worker -o jsonpath='{.items[2].spec.template.spec.providerSpec.value.zone}')-machineset.yml

# Loop through the different worker manifest files
for filename in worker-*-machineset.yml; do
  # Cleanup the manifest files
  yq eval -i 'del(.status,.metadata.annotations,.metadata.creationTimestamp,.metadata.generation,.metadata.uid,.metadata.resourceVersion,.spec.template.spec.taints)' $filename

  # Copy to new GPU named files
  cp $filename ${filename/worker/gpu}
done

# Loop through the GPU manifest files
for filename in gpu-*-machineset.yml; do
  # Replace the worker keywords for metadata
  sed -i -e 's/-worker-/-gpu-/g' $filename

  # Modify the machine type
  # may need to request Subscription quota increase eg for "Standard NCSv2" - note that the value is vCPU based not instance count
  yq eval -i '.spec.template.spec.providerSpec.value.vmSize = "Standard_NC12s_v3"' $filename

  # Update replica count of the nodes
  yq eval -i '.spec.replicas = 1' $filename

  # Create the MachineSets in the cluster
  oc apply -f $filename
done
```