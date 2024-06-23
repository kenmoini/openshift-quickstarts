# Application backup with OpenShift API for Data Protection (OADP)

OADP can backup objects in the cluster that relate to your application workloads - this can be the YAML manifest objects and/or persistent volumes.  Note that OADP does not provide etcd backups.

While OADP can be used to create and restore backups of applications, it's a good practice to leverage GitOps mechanisms to maintain application state and configuration.

The process requires an S3 bucket to backup applications and their data to - you can find a [deploy-minio.md](./deploy-minio.md) file in this directory to deploy a Minio instance for testing purposes.

Additional documentation can be found here: https://docs.openshift.com/container-platform/4.15/backup_and_restore/application_backup_and_restore/oadp-intro.html

## Deploy an Application

The following processes need an application or two to backup.  In the `sample-apps` directory of this repo you can find two applications to test with, `infinite-mario` which is a simple stateless application and `test-mongodb` which uses persistent storage.  The following processes will demonstrate how to backup the manifest objects of both as well as the data of the MongoDB deployment.

## Install the OADP Operator

Begin by installing the OADP Operator - this can be done via the Operator Hub in the Web UI or via the following manifest objects:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-adp
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-adp
  namespace: openshift-adp
spec:
  targetNamespaces:
    - openshift-adp
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: redhat-oadp-operator
  namespace: openshift-adp
spec:
  channel: stable-1.3
  installPlanApproval: Automatic
  name: redhat-oadp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

## Create the S3 Secret

Before deploying the Operator Instances, you need to create a Secret with the Access Key ID and Access Secret:

```yaml
---
kind: Secret
apiVersion: v1
metadata:
  name: cloud-credentials
  namespace: openshift-adp
stringData:
  cloud: |
    [default]
    aws_access_key_id=SOME_KEY_ID
    aws_secret_access_key=SOME_SECRET
```

## Create the DataProtectionApplication

With the Operator installed, you can configure the DataProtectionApplication object that deploys and configures the underlying Velero service:

```yaml
---
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: velero
  namespace: openshift-adp
spec:
  backupLocations:
    - name: default
      velero:
        config:
          profile: "default" # name of the profile created in the Secret above
          insecureSkipTLSVerify: 'true' # if you want TLS validation provide the Root CA signing the S3 service with the objectStorage.caCert spec below
          region: "the-moon" # If this is not a cloud region, give it any sort of name
          s3ForcePathStyle: 'true' # Only required for non-AWS S3 storage
          s3Url: 'http://deep-thought.kemo.labs:9000' # Endpoint URL
        credential: # name and data key of the Secret created above
          name: cloud-credentials
          key: cloud
        default: true
        objectStorage:
          bucket: "ocp-oadp" # bucket name
          prefix: "velero" # folder prefix in the bucket
          # caCert: <base64_encoded_cert_string>
        provider: aws # Denotes S3-compatible storage
  snapshotLocations:
    - velero:
        config:
          profile: "default" # name of the backupLocation above
          region: "the-moon" # Can be anything for non-cloud S3 storage
        provider: "aws" # Denotes S3-compatible storage
  configuration: # Service configuration
    nodeAgent:
      enable: true
      uploaderType: kopia
    velero:
      featureFlags:
        - EnableCSI
      defaultPlugins:
        - aws
        - csi
        - kubevirt
        - openshift
```

## Configure Storage VolumeSnapshotClass

Before backing up application data, your CSI needs to support Volume Snapshots.  If it does, you should have a VolumeSnapshotClass associated with it.

Label the VolumeSnapshotClass with `velero.io/csi-volumesnapshot-class: 'true'`

## Create a Backup CR - App Without Storage

Assuming you're testing with the `infinite-mario` sample application, you can create the following Backup manifest to ship all the objects in that Namespace to S3:

```yaml
---
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: mario-backup
  labels:
    velero.io/storage-location: default # name of the backupLocation defined in the DPA
  namespace: openshift-adp
spec:
  includedNamespaces:
    - infinite-mario
  storageLocation: default # name of the backupLocation defined in the DPA
  ttl: 720h0m0s
  hooks: {} # https://docs.openshift.com/container-platform/4.15/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/oadp-creating-backup-hooks-doc.html
  #includedResources: [] 
  #excludedResources: []
```

You may optionally include or exclude resources, set labelSelectors for the resources to be backed up, and even run commands in a container before/after the backup process is run.

Once the Backup CR has shown that the backup is completed, you should see the packaged resources in the S3 Bucket under `velero/backups/mario-backup`.  You can check the status from the command line via `oc get backup -n openshift-adp mario-backup -o jsonpath='{.status.phase}'`

**Note**: If you delete a Backup CR and the backup remains in the S3 bucket, the Backup CR will be recreated in OpenShift.  To permanently delete a backup you must either delete the data from the S3 bucket, or use the `velero backup delete <backup_CR_name> -n openshift-adp` command.

## Create a Backup CR - App With Storage

Assuming you're testing with the `test-mongodb` sample application, you can create the following Backup manifest to ship all the objects in that Namespace as well as the persistent storage used by the database.

Before performing the backup, add some non-db random data in the PVC with `head -c 107374182 </dev/urandom > /data/db/randomfile`

```yaml
---
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: mongodb-backup
  labels:
    velero.io/storage-location: "default" # name of the backupLocation defined in the DPA
  namespace: openshift-adp
spec:
  includedNamespaces:
    - test-mongodb
  storageLocation: "default" # name of the backupLocation defined in the DPA
  ttl: 720h0m0s
  defaultVolumesToFsBackup: true # Backup storage with Kopia
  snapshotMoveData: true # Backup Snapshots
  hooks: {} # https://docs.openshift.com/container-platform/4.15/backup_and_restore/application_backup_and_restore/backing_up_and_restoring/oadp-creating-backup-hooks-doc.html
  #includedResources: [] 
  #excludedResources: []
```

Once the Backup CR has shown that the backup is completed, you should see the packaged resources in the S3 Bucket under `velero/backups/mongodb-backup` and `velero/kopia/mongodb-backup`.  You can check the status from the command line via `oc get backup -n openshift-adp mongodb-backup -o jsonpath='{.status.phase}'`

## Create a Restore CR - App Without Storage

Assuming that you're restoring the `infinite-mario` application, ensure the `infinite-mario` Namespace is deleted after being backed up.  Then apply the following Restore CR which will apply the Backup to the cluster:

```yaml
---
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: mario-restore
  namespace: openshift-adp
spec:
  backupName: mario-backup
  includedResources: [] 
  excludedResources:
    - nodes
    - events
    - events.events.k8s.io
    - backups.velero.io
    - restores.velero.io
    - resticrepositories.velero.io
    - csinodes.storage.k8s.io
    - volumeattachments.storage.k8s.io
    - backuprepositories.velero.io
```

Once the Restore CR has entered a Completed state, you should be able to find the `infinite-mario` application in its namespace restored as it was before.

## Create a Restore CR - App With Storage

To test restoring an application and its data, first delete the namespace that the application lives in - assuming you're using the `test-mongodb` application, delete the `test-mongodb` namespace.

Once the cluster does not have the workload actively running, create the following Restore CR:

```yaml
---
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: mongodb-restore
  namespace: openshift-adp
spec:
  backupName: mongodb-backup
  includedResources: [] 
  excludedResources:
    - nodes
    - events
    - events.events.k8s.io
    - backups.velero.io
    - restores.velero.io
    - resticrepositories.velero.io
  restorePVs: true # Restore PVCs
```

Once you have applied that Restore CR and it has entered a Completed state, you should see the database and its PVC data restored.  Assuming you created a random file, you can check to make sure that it is still there.

## Scheduled Backups

You can optionally create Scheduled Backups.  The example below will take a backup every day at 7AM:

```yaml
---
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: mongodb-backup
  namespace: openshift-adp
spec:
  schedule: 0 7 * * * 
  template:
    hooks: {}
    includedNamespaces:
      - test-mongodb
    storageLocation: "default" # name of the backupLocation defined in the DPA
    defaultVolumesToFsBackup: true # Backup storage with Kopia
    snapshotMoveData: true # Backup Snapshots
    ttl: 720h0m0s
```