# Deploying Vault to OpenShift

At times it can be useful to have a Vault instance running in OpenShift for testing purposes.  The instructions below will show you how to do that.

## Deploy Vault with Helm

```bash=
## Install Helm locally on your bastion/computer: https://github.com/helm/helm#install

## Add the Hashicorp Helm repo
helm repo add hashicorp https://helm.releases.hashicorp.com

## Update Helm
helm repo update

## Create/switch to a new namespace for Vault
oc new-project vault

## Deploy the Vault Helm Chart
helm install vault hashicorp/vault \
  --namespace vault \
  --set "global.openshift=true" \
  --set "server.dev.enabled=true"
```

## Initialize Vault

Vault comes in a sealed state that locks it down so you need to initialize/unseal the Vault deployment.  Before the Vault pod will enter a Ready state it must first be initialized.

This is done once when freshly deployed:

```bash
## Switch to the Vault project
oc project vault

## Initialize the Vault
oc exec -it vault-0 -- vault operator init -key-shares=1 -key-threshold=1

## Take note of the Vault Key and Root Token!

## Unseal with the Vault Key
oc exec -it vault-0 -- vault operator unseal "$KEY"

## Log in to Vault with the Root Token
oc exec -it vault-0 -- vault login "${ROOT_TOKEN}"

## Enable K8s authentication provider in Vault
oc exec -it vault-0 -- vault auth enable kubernetes

## Enable key/value Secrets Engine
oc exec -it vault-0 -- vault secrets enable -version=2 kv

### Optional - Enable Vault PKI Secrets Engine
oc exec -it vault-0 -- vault secrets enable pki

### Optional - Configure a 10yr max lease time for PKI
oc exec -it vault-0 -- vault secrets tune -max-lease-ttl=87600h pki
```

### Bonus: Deploying a Route for the Vault UI

```bash

## Switch to the Vault project
oc project vault

## Create the Route
cat > vault-ui-route.yaml <<EOF
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: vault-ui
spec:
  to:
    kind: Service
    name: vault
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None
EOF

## Apply the Route
oc apply -f vault-ui-route.yaml

## Get the Route URL
VAULT_UI_ROUTE="https://$(oc get route vault-ui --output=jsonpath='{.status.ingress[0].host}')"
echo $VAULT_UI_ROUTE
```

> At this point, you would populate your Vault with secrets!