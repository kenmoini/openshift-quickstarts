# Node Configuration - Set core User Password

Typically the `core` user should be authenticated to via SSH keys.  In some instances you may need direct console access to log in as the `core` user.  The following MachineConfig can set the password to whatever you'd like:

First create a password hash with `mkpasswd -m SHA-512 superSecurePassword`

Then insert it into the last line in the MachineConfig below:

```yaml
---
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: set-core-userpw-worker
spec:
  config:
    ignition:
      version: 3.2.0
    passwd:
      users:
      - name: core 
        passwordHash: passwordHashHere
```