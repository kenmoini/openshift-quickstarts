# Sample Application - Test MongoDB

To perform test of storage backed workloads you can quickly deploy a MongoDB instance.  It should create about 350MB of used space on the PVC.  The following deployment assumes use of a default StorageClass, if there is no default or you want to use a specific one then modify the `05_pvc.yml` to match your environment.

```bash
# Deploy the database - assuming this is the current directory
oc apply -k manifests
```

Some examples such as OADP Backups will use this application.