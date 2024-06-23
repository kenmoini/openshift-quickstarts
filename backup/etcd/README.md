# etcd Backup & Restore

etcd runs on the control plane nodes of your cluster and contains the cluster state.  It is advised to perform a backup before and after upgrades and even regularly.  A few key things to note:

- Back up your cluster’s etcd data regularly and store in a secure location ideally outside the OpenShift Container Platform environment.
- Do not take an etcd backup before the first certificate rotation completes, which occurs 24 hours after installation, otherwise the backup will contain expired certificates.
- It is recommended to take etcd backups during non-peak usage hours because the etcd snapshot has a high I/O cost.
- Be sure to take an etcd backup after you upgrade your cluster. This is important because when you restore your cluster, you must use an etcd backup that was taken from the same z-stream release. For example, an OpenShift Container Platform 4.15.4 cluster must use an etcd backup that was taken from 4.15.4.
- Run the backup procedure from only one of the control plane nodes.
- You must have cluster-admin access to backup etcd
- If using an outbound proxy, make sure to export the `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` environmental variables once you have changed into the host root file system

Additional documentation can be found here:

- [Backing up etcd](https://docs.openshift.com/container-platform/4.15/backup_and_restore/control_plane_backup_and_restore/backing-up-etcd.html)
- [Restoring etcd](https://docs.openshift.com/container-platform/4.15/backup_and_restore/control_plane_backup_and_restore/disaster_recovery/scenario-2-restoring-cluster-state.html)

## Backup etcd

```bash
# Start a debug terminal on one of the control plane nodes - use `oc get nodes` to find the name/role
oc debug --as-root node NODE_NAME

# Once inside the debug terminal, change to the host file system
chroot /host

# Run the backup script
/usr/local/bin/cluster-backup.sh /home/core/assets/backup
```

After running the backup script, you'll find two files, `snapshot_<datetimestamp>.db` and `static_kuberesources_<datetimestamp>.tar.gz`.  Copy these files to an external location, can easily be done via the `scp` command.

## Restoring etcd

You can restore etcd to a previous state, so long as the backup is not too old to invalidate the dates on the cluster certificates.  The process essentially sends a cluster "back in time" which will cause it to take a moment while NTP/PTP, cluster operators, networking, and storage reach a consistent quorum.  Important notes:

- Once the restoration process has started, it cannot be stoped and the API will become inaccessible.
- This should be done as a last resort in case repairing or redeployment is not an option.
- The process requires SSH access to all healthy control plane hosts.
- You must have cluster-admin access via a certificate-based kubeconfig file, such as the one used during installation.
- The filenames must not have been changed from what was set during the backup procedure.
- One control plane host will be used for the restoration process, the other control plane hosts will need to have their static pods stopped.
- The restore process can cause nodes to enter the NotReady state if the node certificates were updated after the last etcd backup.
- If your OpenShift Container Platform cluster uses persistent storage of any form, a state of the cluster is typically stored outside etcd. It might be an Elasticsearch cluster running in a pod or a database running in a StatefulSet object. When you restore from an etcd backup, the status of the workloads in OpenShift Container Platform is also restored. However, if the etcd snapshot is old, the status might be invalid or outdated.
- The contents of persistent volumes (PVs) are never part of the etcd snapshot. When you restore an OpenShift Container Platform cluster from an etcd snapshot, non-critical workloads might gain access to critical data, or vice-versa.
- If you are using the OVN-Kubernetes network plugin, you must restart the Open Virtual Network (OVN) Kubernetes pods on all the nodes one by one.

> The restoration process is involved and complicated - it is best to read and follow the instructions here: https://docs.openshift.com/container-platform/4.15/backup_and_restore/control_plane_backup_and_restore/disaster_recovery/scenario-2-restoring-cluster-state.html
