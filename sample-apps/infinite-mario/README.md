# Sample Application - Infinite Mario

Infinite Mario is a randomly generating Super Mario Bros clone that can be played in your browser.  This is a stateless application that can easily be deployed to test functions in OpenShift.

```bash
# Deploy the game - assuming this is the current directory
oc apply -k manifests
```

Some examples such as OADP Backups will use this application.