# Networking - MetalLB Deployment

**User Story:** I need to expose applications via an IP and have its traffic load balanced with no other external dependencies.

In cloud environments you can create LoadBalancer type Services and often have the cloud provider integration automatically create a Load Balancer that points to the cluster.

In on-premise environments where there are no such available services, you can use MetalLB to create an in-cluster Load Balancer service.  This allows exposing Services on non HTTP/HTTPS endpoints that the Route Ingress typically would handle.  This is useful for things such as exposing databases or other TCP/UDP workloads.

The following example assumes the use of Layer 2 mode in MetalLB - BGP configuration should be referenced from the documentation.

Additional documentation can be found here: https://docs.openshift.com/container-platform/4.15/networking/metallb/about-metallb.html

## Installing the MetalLB Operator

You can install the MetalLB Operator via the Web UI and the Operator Hub, or via the following manifest objects:

```yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: metallb-system
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec: {}
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: metallb
  namespace: metallb-system
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec: {}
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: metallb-operator
  namespace: metallb-system
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  channel: stable
  installPlanApproval: Automatic
  name: metallb-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

## Deploying the MetalLB Instance

With the Operator installed, you can now deploy the MetalLB system instance which will deploy the various speaker pods.

```yaml
---
apiVersion: metallb.io/v1beta1
kind: MetalLB
metadata:
  name: metallb
  namespace: metallb-system
  annotations:
    argocd.argoproj.io/sync-wave: "10"
spec: {}
  #logLevel: debug

  # Use a nodeSelector if you want MetalLB to only run on certain nodes
  #nodeSelector:
  #  node-role.kubernetes.io/worker: ""

  # Resource Limits
  #controllerConfig:
  #  resources:
  #    limits:
  #      cpu: "200m"
  #speakerConfig:
  #  resources:
  #    limits:
  #      cpu: "300m"
```

Once the instance has been deployed, you should see the Speaker Pods launched in the `metallb-system` Namespace.

Additional configuration options can be found here: https://docs.openshift.com/container-platform/4.15/networking/metallb/metallb-operator-install.html#nw-metallb-operator-deployment-specifications-for-metallb_metallb-operator-install

## Configuring an IP Address Pool

With MetalLB installed, you can proceed to create a set of manifest objects that will make blocks of IPs available for use by LoadBalancer type Services.  You can have multiple address segments in an AddressPool and multiple AddressPools for different Namespaces.

```yaml
---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: lab-pool
  namespace: metallb-system
  labels:
    zone: east
spec:
  addresses:
    - 192.168.100.0/24 # full subnet
    - 192.168.200.10/32 # single IP
    - 192.168.255.1-192.168.255.5 # range of IPs
    - 2002:2:2::1-2002:2:2::100 # IPv6 network range
  autoAssign: false # highly recommended if you have a small number of IPs or want to manage assignments
  avoidBuggyIPs: true # avoid allocating x.y.z.0 and x.y.z.255

# Optional configuratoin
  serviceAllocation:
    # Make these IPs available in a subset of Namespaces
    namespaces:
      - namespace-a
      - namespace-b
    # Make these IPs available to Namespaces with a specific label
    namespaceSelectors: 
      - matchLabels:
          zone: east
    # Make these IPs available to Services matching specific labels
    serviceSelectors: 
      - matchExpressions:
        - key: app
          operator: In
          values:
            - blue-app
            - green-app
    # Defines the priority between IP address pools when more than one IP address pool matches a service or namespace.
    # A lower number indicates a higher priority.
    priority: 50
```

## Creating an L2Advertisement

With the IPAddressPool created, you can advertise its availability to nodes in your cluster and on specific interfaces.

```yaml
---
kind: L2Advertisement
apiVersion: metallb.io/v1beta1
metadata:
  name: lab-l2-adv
  namespace: metallb-system
spec:
  ipAddressPools: # used for targeting specific IPAddressPools
    - lab-pool # Must match the name of the IPAddressPool
  # ipAddressPools and ipAddressPoolSelectors are mutually exclusive - use one or the other
  ipAddressPoolSelectors: # used to target groups of IPAddressPools by labels
    - matchExpressions:
        - key: zone
          operator: In
          values:
            - east
   interfaces: # what interface to use
     - bond1
```

## LoadBalancer type Service Testing

Once both an IPAddressPool and L2Advertisement are defined you can continue to test the integration by deploying a simple workload and Service:

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd
spec:
  selector:
    matchLabels:
      app: httpd
  replicas: 1
  template:
    metadata:
      labels:
        app: httpd
    spec:
      containers:
        - name: httpd
          resources:
            limits:
              cpu: 100m
              memory: 128Mi
            requests:
              cpu: 100m
              memory: 128Mi
          image: registry.access.redhat.com/ubi8/httpd-24@sha256:b72f2fd69dbc32d273bebb2da30734c9bc8d9acfd210200e9ad5e69d8b089372
          ports:
            - containerPort: 8080
```

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: httpd
  annotations:
    # If your IPAddressPool does not autoAssign IPs, use these annotations
    # Make sure the annotations match the specification fo your IPAddressPool
    metallb.universe.tf/address-pool: lab-pool
    metallb.universe.tf/loadBalancerIPs: 192.168.70.11
spec:
  selector:
    app: httpd
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: LoadBalancer
```

Once you have the Deployment and Service created, you can test the MetalLB assigned LoadBalancer IP by `curl`ing the Service's LoadBalancer IP or navigating to it in your web browser at port `8080`.
