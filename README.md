# OpenShift Quickstarts

This repository has a collection of walk-throughs that address common patterns when deploying OpenShift.

Note that these are somewhat opinionated guides, ideal for POCs, and that production deployments may need additional consideration.

## Index

> Non-linked items will soon be added

### AI/ML

- [NVIDIA GPU Operator Setup](./ai-ml/nvidia-gpu-operator-setup.md)
- OpenShift AI Quickstart
- StableDiffusion on OpenShift
- IBM Granite on OpenShift
- Model Conversion and Serving
- Darknet/YOLO on OpenShift

### Alerting/Monitoring

- AlertManager Configuration
- Custom AlertManager Rules
- Custom Prometheus Monitors
- Custom & Debug Receivers
- Exporting Prometheus Metrics

### Backup

- [OADP Operator (Velero)](./backup/oadp/)
- [etcd Backup](./backup/etcd/)

### GitOps

- GitOps Quickstart
- Advanced Cluster Management Integration
- App of Apps Pattern
- ApplicationSet Pattern
- Kustomize Application
- Helm Application
- Application with PolicyGenerators

### Identity Management

- Cluster-wide Root CA Trust Bundle
- HTPassword Authentication
- LDAP Integration
- Active Directory Integration
- Google Authentication
- OIDC Authentication
- SAML Authentication

### Image Registries

- [Cluster Image Registry Configuration](./registry-configuration/)
- [Internal Image Registry Deployment](./internal-registry/)

### Logging

- [OpenShift Logging, forwarding to Elasticstack](./logging/external-elasticstack/)
- OpenShift Logging, forwarding to Splunk

### MachineConfig

- [SystemD Services](./node-config/machineconfigs/systemd-services.md)
- [Set Kubelet Configuration](./node-config/machineconfigs/kubelet-configuration.md)
- [Set `core` User Password](./node-config/machineconfigs/set-core-user-password.md)

### MachineSets

- [AWS GPU MachineSets](./node-config/machinesets/aws-gpu-machineset.md)
- [Azure GPU MachineSets](./node-config/machinesets/azure-gpu-machineset.md)

### Networking

- [Installing NMState Operator](./networking/nmstate-operator.md)
- [Defining Additional Bonds, Bridges, and VLANs](./networking/nmstate-examples.md)
- [OVS Bridge to Management Network](./networking/ovs-bridge-mapping.md)
- [Live Migration Network for OpenShift Virtualization](./networking/live-migration-network.md)
- [MetalLB Deployment](./networking/metallb-deployment.md)

### Secrets Management

- [External Secrets Operator and Hashicorp Vault](./external-secrets-vault/)

### Sample Applications

- [Infinite Mario (Stateless)](./sample-apps/infinite-mario/)
- [Simple Apache HTTP Service (Stateless)](./sample-apps/httpd/)
- [Test Mongo Database (Stateful)](./sample-apps/test-mongodb/)