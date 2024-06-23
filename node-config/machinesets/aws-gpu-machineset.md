# MachineSets - Creating a GPU MachineSet - AWS

If you are running OpenShift in AWS you can run the following commands to copy down the default worker MachineSet and make some alterations to create a new MachineSet for adding GPU nodes to the cluster.

Find your desired EC2 instance type here: https://aws.amazon.com/ec2/instance-types/#Accelerated_Computing

You may need to request quota increase eg for "Running On-Demand P instances" - note that the value is vCPU based not instance count.

**Note**:  If you are running the managed Red Hat OpenShift Service on AWS (ROSA) then you will need to make the MachineSet via the [Red Hat Hybrid Cloud Console](https://console.redhat.com).  You could also make it manually with the following manifest object but to do so you'll need to remove `machine.openshift.io` from the sre-regular-user-validation ValidatingWebhookConfiguration which is not supported and would be added back in short order automatically by the service.

## Single AZ Cluster

The following instructions assume a single AZ cluster - for a multi-AZ cluster you would need to create/modify multiple MachineSets.

```bash
# Get the Worker MachineSet and save it to a file
oc get machineset -n openshift-machine-api $(oc get machineset -n openshift-machine-api --selector hive.openshift.io/machine-pool=worker -o jsonpath='{.items[0].metadata.name}') -o yaml > worker-machineset.yml

# Cleanup the object
yq eval -i 'del(.status,.metadata.annotations,.metadata.creationTimestamp,.metadata.generation,.metadata.uid,.metadata.resourceVersion,.spec.template.spec.taints)' worker-machineset.yml

# Copy to a new file
cp worker-machineset.yml gpu-machineset.yml

# Replace the worker keywords
sed -i -e 's/worker/gpu/g' gpu-machineset.yml

# Reset some values back to intended worker assets
sed -i -e 's/gpu-profile/worker-profile/g' gpu-machineset.yml
sed -i -e 's/gpu-user-data/worker-user-data/g' gpu-machineset.yml
sed -i -e 's/gpu-sg/worker-sg/g' gpu-machineset.yml

# Modify the machine type
yq eval -i '.spec.template.spec.providerSpec.value.instanceType = "p3.8xlarge"' gpu-machineset.yml

# update metadata
yq -i '.spec.template.spec.providerSpec.value.tags[] |= select(.name == "red-hat-managed").value = "false"' gpu-machineset.yml

# Update replica count of the nodes
yq eval -i '.spec.replicas = 1' gpu-machineset.yml

# Create the machineset
oc apply -f gpu-machineset.yml
```

## Multi AZ Cluster

```bash
# Get the Worker MachineSets and save them to a file
oc get machineset -n openshift-machine-api $(oc get machineset -n openshift-machine-api --selector hive.openshift.io/machine-pool=worker -o jsonpath='{.items[0].metadata.name}') -o yaml > $(oc get machineset -n openshift-machine-api --selector hive.openshift.io/machine-pool=worker -o jsonpath='{.items[0].spec.template.spec.providerSpec.value.placement.availabilityZone}')-machineset.yml

oc get machineset -n openshift-machine-api $(oc get machineset -n openshift-machine-api --selector hive.openshift.io/machine-pool=worker -o jsonpath='{.items[1].metadata.name}') -o yaml > $(oc get machineset -n openshift-machine-api --selector hive.openshift.io/machine-pool=worker -o jsonpath='{.items[1].spec.template.spec.providerSpec.value.placement.availabilityZone}')-machineset.yml

oc get machineset -n openshift-machine-api $(oc get machineset -n openshift-machine-api --selector hive.openshift.io/machine-pool=worker -o jsonpath='{.items[2].metadata.name}') -o yaml > $(oc get machineset -n openshift-machine-api --selector hive.openshift.io/machine-pool=worker -o jsonpath='{.items[2].spec.template.spec.providerSpec.value.placement.availabilityZone}')-machineset.yml

# Loop through the different worker manifest files
for filename in worker-*-machineset.yml; do
  # Cleanup the manifest files
  yq eval -i 'del(.status,.metadata.annotations,.metadata.creationTimestamp,.metadata.generation,.metadata.uid,.metadata.resourceVersion,.spec.template.spec.taints)' $filename

  # Copy to new GPU named files
  cp $filename ${filename/worker/gpu}
done

# Loop through the GPU manifest files
for filename in gpu-*-machineset.yml; do
  # Replace the worker keywords
  sed -i -e 's/worker/gpu/g' $filename

  # Reset some values back to intended worker assets
  sed -i -e 's/gpu-profile/worker-profile/g' $filename
  sed -i -e 's/gpu-user-data/worker-user-data/g' $filename
  sed -i -e 's/gpu-sg/worker-sg/g' $filename

  # Modify the machine type
  yq eval -i '.spec.template.spec.providerSpec.value.instanceType = "p3.8xlarge"' $filename

  # update metadata
  yq -i '.spec.template.spec.providerSpec.value.tags[] |= select(.name == "red-hat-managed").value = "false"' $filename

  # Update replica count of the nodes
  yq eval -i '.spec.replicas = 1' $filename

  # Create the machineset
  oc apply -f $filename
done
```