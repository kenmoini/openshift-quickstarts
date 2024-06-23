# AI/ML - NVIDIA GPU Operator Setup

In order to use NVIDIA GPUs on OpenShift you must first setup a couple Operators and configure them.  Of course you also need to have nodes with GPUs available.

If you're running a cluster in AWS/Azure you can easily add a new MachineSet that will add some GPU-enabled nodes to your cluster.  You can find examples of creating those MachineSets in the [node-config/machinesets/](../node-config/machinesets/) directory of this repo.

## Install the Node Feature Discovery Operator

Before installing the NVIDIA GPU Operator you must install the NFD Operator.  This operator is responsible for detecting node hardware and configuration and applying annotations and labels to the Nodes in the cluster.  These annotations and labels are needed for the NVIDIA GPU Operator to know what nodes to enable.

You can install the NFD Operator from the Web UI via the Operator Hub or with the following manifest objects:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    openshift.io/display-name: "Node Feature Discovery Operator"
  labels:
    openshift.io/cluster-monitoring: 'true'
  name: openshift-nfd
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: nfd-group
  namespace: openshift-nfd
spec:
  targetNamespaces:
    - openshift-nfd
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: nfd
  namespace: openshift-nfd
spec:
  channel: stable
  installPlanApproval: Automatic
  name: nfd
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

## Instantiate the NFD Operator

Once the NFD Operator is installed, you must create the NodeFeatureDiscovery CR that is now made available by the operator.  This will actually deploy the NFD DaemonSet to perform the detection and labeling process:

```yaml
---
apiVersion: nfd.openshift.io/v1
kind: NodeFeatureDiscovery
metadata:
  name: nfd-instance
spec:
  operand:
    image: registry.redhat.io/openshift4/ose-node-feature-discovery:latest
    servicePort: 12000
  workerConfig:
    configData: |
      core:
        sleepInterval: 60s
      sources:
        pci:
          deviceClassWhitelist:
            - "0200"
            - "03"
            - "12"
          deviceLabelFields:
            - "vendor"
```

You can also create the default specified NodeFeatureDiscovery CR via the Web UI to see many more available configuration options.  This should be fine as a default.

## Install the NVIDIA GPU Operator

Once the NodeFeatureDiscovery instance has been created and moved to an "Available" state you can continue to install the NVIDIA GPU Operator.  This could be done by clicking through the Operator Hub, or via the following manifests:

```yaml
---
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
    openshift.io/display-name: "NVIDIA GPU Operator"
  labels:
    openshift.io/cluster-monitoring: 'true'
  name: nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: gpu-operator-certified-group
  namespace: nvidia-gpu-operator
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  targetNamespaces:
    - nvidia-gpu-operator
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: gpu-operator-certified
  namespace: nvidia-gpu-operator
  annotations:
    argocd.argoproj.io/sync-wave: "3"
spec:
  channel: stable
  installPlanApproval: Automatic
  name: gpu-operator-certified
  source: certified-operators
  sourceNamespace: openshift-marketplace
```

## Create the NVIDIA GPU Operator ClusterPolicy

With the GPU Operator installed you can create the ClusterPolicy CR that will actually go out and deploy the needed Pods to enable GPU workloads on the cluster.  This is where you'd enable MIG, Time Slicing, and vGPUs if you have nodes that support those features.

Note that Time Slicing is supported by all cards however it does not have memory isolation which can lead to CUDA enabled workloads to fail if resource contention becomes an issue.

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: device-plugin-config
  namespace: nvidia-gpu-operator
  annotations:
    argocd.argoproj.io/sync-wave: "10"
data: {}
---
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
  namespace: nvidia-gpu-operator
  annotations:
    argocd.argoproj.io/sync-wave: "15"
spec:
  operator:
    defaultRuntime: crio
    use_ocp_driver_toolkit: true
    initContainer: {}
  sandboxWorkloads:
    enabled: false
    defaultWorkload: container
  driver:
    enabled: true
    upgradePolicy:
      autoUpgrade: true
      drain:
        deleteEmptyDir: false
        enable: false
        force: false
        timeoutSeconds: 300
      maxParallelUpgrades: 1
      maxUnavailable: 25%
      podDeletion:
        deleteEmptyDir: false
        force: false
        timeoutSeconds: 300
      waitForCompletion:
        timeoutSeconds: 0
    repoConfig:
      configMapName: ''
    certConfig:
      name: ''
    licensingConfig:
      nlsEnabled: false
      configMapName: ''
    virtualTopology:
      config: ''
    kernelModuleConfig:
      name: ''
  dcgmExporter:
    enabled: true
    #config:
    #  name: 'console-plugin-nvidia-gpu'
    serviceMonitor:
      enabled: true
  dcgm:
    enabled: true
  daemonsets:
    updateStrategy: RollingUpdate
    rollingUpdate:
      maxUnavailable: '1'
  devicePlugin:
    enabled: true
    config:
      name: ''
      default: ''
  gfd:
    enabled: true
  migManager:
    enabled: true
  nodeStatusExporter:
    enabled: true
  mig:
    strategy: single
  toolkit:
    enabled: true
  validator:
    plugin:
      env:
        - name: WITH_WORKLOAD
          value: 'true'
  vgpuManager:
    enabled: false
  vgpuDeviceManager:
    enabled: true
  sandboxDevicePlugin:
    enabled: true
  vfioManager:
    enabled: true
  gds:
    enabled: false
```

- For **Time Slicing** configuration, see the NVIDIA GPU Operator documentation: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-sharing.html
- For **MIG enablement** see the NVIDIA GPU Operator documentation: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-mig.html
- For using **GPUs with KubeVirt/OpenShift Virtualization VMs** see this doucmentation: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/gpu-operator-kubevirt.html
- For **vGPU enablement**, see the following documentation: https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/install-gpu-operator-vgpu.html

Once the ClusterPolicy CR has been deployed you should see a number of Pods deployed in the `nvidia-gpu-operator` Namespace.  Once the installation has been completed you should see the `nvidia-cuda-validator-xxxxx` Pod marked with a "Completed" status - this Pod tests GPU CUDA workloads post-install.

## Enable DCGM Metrics Dashboard

The NVIDIA Data Center GPU Manager (DCGM) exports Prometheus metrics that can be displayed on the OpenShift Monitoring dashboard.  To enable that view create the following ConfigMap:

<details>
  <summary>Show ConfigMap (it's very long)</summary>

  ```yaml
  ---
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: nvidia-dcgm-exporter-dashboard
    namespace: openshift-config-managed
    labels:
      console.openshift.io/dashboard: "true"
      console.openshift.io/odc-dashboard: "true"
  data:
    dcgm-exporter-dashboard.json: |
      {
        "__requires": [
          {
            "type": "panel",
            "id": "gauge",
            "name": "Gauge",
            "version": ""
          },
          {
            "type": "grafana",
            "id": "grafana",
            "name": "Grafana",
            "version": "6.7.3"
          },
          {
            "type": "panel",
            "id": "graph",
            "name": "Graph",
            "version": ""
          },
          {
            "type": "datasource",
            "id": "prometheus",
            "name": "Prometheus",
            "version": "1.0.0"
          }
        ],
        "annotations": {
          "list": [
            {
              "$$hashKey": "object:192",
              "builtIn": 1,
              "datasource": "-- Grafana --",
              "enable": true,
              "hide": true,
              "iconColor": "rgba(0, 211, 255, 1)",
              "name": "Annotations & Alerts",
              "type": "dashboard"
            }
          ]
        },
        "description": "This dashboard is to display the metrics from DCGM Exporter on a Kubernetes (1.19+) cluster",
        "editable": true,
        "gnetId": 12239,
        "graphTooltip": 0,
        "id": null,
        "iteration": 1588401887165,
        "links": [],
        "panels": [
          {
            "aliasColors": {},
            "bars": false,
            "dashLength": 10,
            "dashes": false,
            "datasource": "$datasource",
            "fill": 1,
            "fillGradient": 0,
            "gridPos": {
              "h": 8,
              "w": 18,
              "x": 0,
              "y": 0
            },
            "hiddenSeries": false,
            "id": 12,
            "legend": {
              "alignAsTable": true,
              "avg": true,
              "current": true,
              "max": true,
              "min": false,
              "rightSide": true,
              "show": true,
              "total": false,
              "values": true
            },
            "lines": true,
            "linewidth": 2,
            "nullPointMode": "null",
            "options": {
              "dataLinks": []
            },
            "percentage": false,
            "pointradius": 2,
            "points": false,
            "renderer": "flot",
            "seriesOverrides": [],
            "spaceLength": 10,
            "stack": false,
            "steppedLine": false,
            "targets": [
              {
                "expr": "DCGM_FI_DEV_GPU_TEMP{instance=~\"$instance\", gpu=~\"$gpu\"}",
                "instant": false,
                "interval": "",
                "legendFormat": "GPU {{gpu}}",
                "refId": "A"
              }
            ],
            "thresholds": [],
            "timeFrom": null,
            "timeRegions": [],
            "timeShift": null,
            "title": "GPU Temperature",
            "tooltip": {
              "shared": true,
              "sort": 0,
              "value_type": "individual"
            },
            "type": "graph",
            "xaxis": {
              "buckets": null,
              "mode": "time",
              "name": null,
              "show": true,
              "values": []
            },
            "yaxes": [
              {
                "format": "celsius",
                "label": null,
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              },
              {
                "format": "short",
                "label": null,
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              }
            ],
            "yaxis": {
              "align": false,
              "alignLevel": null
            }
          },
          {
            "datasource": "$datasource",
            "gridPos": {
              "h": 8,
              "w": 6,
              "x": 18,
              "y": 0
            },
            "id": 14,
            "options": {
              "fieldOptions": {
                "calcs": [
                  "mean"
                ],
                "defaults": {
                  "color": {
                    "mode": "thresholds"
                  },
                  "mappings": [],
                  "max": 100,
                  "min": 0,
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": null
                      },
                      {
                        "color": "#EAB839",
                        "value": 83
                      },
                      {
                        "color": "red",
                        "value": 87
                      }
                    ]
                  },
                  "unit": "celsius"
                },
                "overrides": [],
                "values": false
              },
              "orientation": "auto",
              "showThresholdLabels": false,
              "showThresholdMarkers": true
            },
            "pluginVersion": "6.7.3",
            "targets": [
              {
                "expr": "avg(DCGM_FI_DEV_GPU_TEMP{instance=~\"$instance\", gpu=~\"$gpu\"})",
                "interval": "",
                "legendFormat": "",
                "refId": "A"
              }
            ],
            "timeFrom": null,
            "timeShift": null,
            "title": "GPU Avg. Temp",
            "type": "gauge"
          },
          {
            "aliasColors": {},
            "bars": false,
            "dashLength": 10,
            "dashes": false,
            "datasource": "$datasource",
            "fill": 1,
            "fillGradient": 0,
            "gridPos": {
              "h": 8,
              "w": 18,
              "x": 0,
              "y": 8
            },
            "hiddenSeries": false,
            "id": 10,
            "legend": {
              "alignAsTable": true,
              "avg": true,
              "current": true,
              "max": true,
              "min": false,
              "rightSide": true,
              "show": true,
              "total": false,
              "values": true
            },
            "lines": true,
            "linewidth": 2,
            "nullPointMode": "null",
            "options": {
              "dataLinks": []
            },
            "percentage": false,
            "pluginVersion": "6.5.2",
            "pointradius": 2,
            "points": false,
            "renderer": "flot",
            "seriesOverrides": [],
            "spaceLength": 10,
            "stack": false,
            "steppedLine": false,
            "targets": [
              {
                "expr": "DCGM_FI_DEV_POWER_USAGE{instance=~\"$instance\", gpu=~\"$gpu\"}",
                "interval": "",
                "legendFormat": "GPU {{gpu}}",
                "refId": "A"
              }
            ],
            "thresholds": [],
            "timeFrom": null,
            "timeRegions": [],
            "timeShift": null,
            "title": "GPU Power Usage",
            "tooltip": {
              "shared": true,
              "sort": 0,
              "value_type": "individual"
            },
            "type": "graph",
            "xaxis": {
              "buckets": null,
              "mode": "time",
              "name": null,
              "show": true,
              "values": []
            },
            "yaxes": [
              {
                "format": "watt",
                "label": null,
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              },
              {
                "format": "short",
                "label": null,
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              }
            ],
            "yaxis": {
              "align": false,
              "alignLevel": null
            }
          },
          {
            "cacheTimeout": null,
            "datasource": "$datasource",
            "gridPos": {
              "h": 8,
              "w": 6,
              "x": 18,
              "y": 8
            },
            "id": 16,
            "links": [],
            "options": {
              "fieldOptions": {
                "calcs": [
                  "sum"
                ],
                "defaults": {
                  "color": {
                    "mode": "thresholds"
                  },
                  "mappings": [],
                  "max": 2400,
                  "min": 0,
                  "nullValueMode": "connected",
                  "thresholds": {
                    "mode": "absolute",
                    "steps": [
                      {
                        "color": "green",
                        "value": null
                      },
                      {
                        "color": "#EAB839",
                        "value": 1800
                      },
                      {
                        "color": "red",
                        "value": 2200
                      }
                    ]
                  },
                  "unit": "watt"
                },
                "overrides": [],
                "values": false
              },
              "orientation": "horizontal",
              "showThresholdLabels": false,
              "showThresholdMarkers": true
            },
            "pluginVersion": "6.7.3",
            "targets": [
              {
                "expr": "sum(DCGM_FI_DEV_POWER_USAGE{instance=~\"$instance\", gpu=~\"$gpu\"})",
                "instant": true,
                "interval": "",
                "legendFormat": "",
                "range": false,
                "refId": "A"
              }
            ],
            "timeFrom": null,
            "timeShift": null,
            "title": "GPU Power Total",
            "type": "gauge"
          },
          {
            "aliasColors": {},
            "bars": false,
            "dashLength": 10,
            "dashes": false,
            "datasource": "$datasource",
            "fill": 1,
            "fillGradient": 0,
            "gridPos": {
              "h": 8,
              "w": 12,
              "x": 0,
              "y": 16
            },
            "hiddenSeries": false,
            "id": 2,
            "interval": "",
            "legend": {
              "alignAsTable": true,
              "avg": true,
              "current": true,
              "max": true,
              "min": false,
              "rightSide": true,
              "show": true,
              "sideWidth": null,
              "total": false,
              "values": true
            },
            "lines": true,
            "linewidth": 2,
            "nullPointMode": "null",
            "options": {
              "dataLinks": []
            },
            "percentage": false,
            "pointradius": 2,
            "points": false,
            "renderer": "flot",
            "seriesOverrides": [],
            "spaceLength": 10,
            "stack": false,
            "steppedLine": false,
            "targets": [
              {
                "expr": "DCGM_FI_DEV_SM_CLOCK{instance=~\"$instance\", gpu=~\"$gpu\"} * 1000000",
                "format": "time_series",
                "interval": "",
                "intervalFactor": 1,
                "legendFormat": "GPU {{gpu}}",
                "refId": "A"
              }
            ],
            "thresholds": [],
            "timeFrom": null,
            "timeRegions": [],
            "timeShift": null,
            "title": "GPU SM Clocks",
            "tooltip": {
              "shared": true,
              "sort": 0,
              "value_type": "individual"
            },
            "type": "graph",
            "xaxis": {
              "buckets": null,
              "mode": "time",
              "name": null,
              "show": true,
              "values": []
            },
            "yaxes": [
              {
                "decimals": null,
                "format": "hertz",
                "label": "",
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              },
              {
                "format": "short",
                "label": null,
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              }
            ],
            "yaxis": {
              "align": false,
              "alignLevel": null
            }
          },
          {
            "aliasColors": {},
            "bars": false,
            "dashLength": 10,
            "dashes": false,
            "datasource": "$datasource",
            "fill": 1,
            "fillGradient": 0,
            "gridPos": {
              "h": 8,
              "w": 12,
              "x": 0,
              "y": 24
            },
            "hiddenSeries": false,
            "id": 6,
            "legend": {
              "alignAsTable": true,
              "avg": true,
              "current": true,
              "max": true,
              "min": false,
              "rightSide": true,
              "show": true,
              "total": false,
              "values": true
            },
            "lines": true,
            "linewidth": 2,
            "nullPointMode": "null",
            "options": {
              "dataLinks": []
            },
            "percentage": false,
            "pointradius": 2,
            "points": false,
            "renderer": "flot",
            "seriesOverrides": [],
            "spaceLength": 10,
            "stack": false,
            "steppedLine": false,
            "targets": [
              {
                "expr": "DCGM_FI_DEV_GPU_UTIL{instance=~\"$instance\", gpu=~\"$gpu\"}",
                "interval": "",
                "legendFormat": "GPU {{gpu}}",
                "refId": "A"
              }
            ],
            "thresholds": [],
            "timeFrom": null,
            "timeRegions": [],
            "timeShift": null,
            "title": "GPU Utilization",
            "tooltip": {
              "shared": true,
              "sort": 0,
              "value_type": "cumulative"
            },
            "type": "graph",
            "xaxis": {
              "buckets": null,
              "mode": "time",
              "name": null,
              "show": true,
              "values": []
            },
            "yaxes": [
              {
                "format": "percent",
                "label": null,
                "logBase": 1,
                "max": "100",
                "min": "0",
                "show": true
              },
              {
                "format": "short",
                "label": null,
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              }
            ],
            "yaxis": {
              "align": false,
              "alignLevel": null
            }
          },
          {
            "aliasColors": {},
            "bars": false,
            "dashLength": 10,
            "dashes": false,
            "datasource": "$datasource",
            "fill": 1,
            "fillGradient": 0,
            "gridPos": {
              "h": 8,
              "w": 12,
              "x": 0,
              "y": 32
            },
            "hiddenSeries": false,
            "id": 18,
            "legend": {
              "alignAsTable": true,
              "avg": true,
              "current": true,
              "max": true,
              "min": false,
              "rightSide": true,
              "show": true,
              "total": false,
              "values": true
            },
            "lines": true,
            "linewidth": 2,
            "nullPointMode": "null",
            "options": {
              "dataLinks": []
            },
            "percentage": false,
            "pointradius": 2,
            "points": false,
            "renderer": "flot",
            "seriesOverrides": [],
            "spaceLength": 10,
            "stack": false,
            "steppedLine": false,
            "targets": [
              {
                "expr": "DCGM_FI_DEV_FB_USED{instance=~\"$instance\", gpu=~\"$gpu\"}",
                "interval": "",
                "legendFormat": "GPU {{gpu}}",
                "refId": "A"
              }
            ],
            "thresholds": [],
            "timeFrom": null,
            "timeRegions": [],
            "timeShift": null,
            "title": "GPU Framebuffer Mem Used",
            "tooltip": {
              "shared": true,
              "sort": 0,
              "value_type": "individual"
            },
            "type": "graph",
            "xaxis": {
              "buckets": null,
              "mode": "time",
              "name": null,
              "show": true,
              "values": []
            },
            "yaxes": [
              {
                "format": "decmbytes",
                "label": null,
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              },
              {
                "format": "short",
                "label": null,
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              }
            ],
            "yaxis": {
              "align": false,
              "alignLevel": null
            }
          },
          {
            "aliasColors": {},
            "bars": false,
            "dashLength": 10,
            "dashes": false,
            "datasource": "$datasource",
            "fill": 1,
            "fillGradient": 0,
            "gridPos": {
              "h": 8,
              "w": 12,
              "x": 0,
              "y": 24
            },
            "hiddenSeries": false,
            "id": 4,
            "legend": {
              "alignAsTable": true,
              "avg": true,
              "current": true,
              "max": true,
              "min": false,
              "rightSide": true,
              "show": true,
              "total": false,
              "values": true
            },
            "lines": true,
            "linewidth": 2,
            "nullPointMode": "null",
            "options": {
              "dataLinks": []
            },
            "percentage": false,
            "pointradius": 2,
            "points": false,
            "renderer": "flot",
            "seriesOverrides": [],
            "spaceLength": 10,
            "stack": false,
            "steppedLine": false,
            "targets": [
              {
                "expr": "DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{instance=~\"$instance\", gpu=~\"$gpu\"}",
                "interval": "",
                "legendFormat": "GPU {{gpu}}",
                "refId": "A"
              }
            ],
            "thresholds": [],
            "timeFrom": null,
            "timeRegions": [],
            "timeShift": null,
            "title": "Tensor Core Utilization",
            "tooltip": {
              "shared": true,
              "sort": 0,
              "value_type": "cumulative"
            },
            "type": "graph",
            "xaxis": {
              "buckets": null,
              "mode": "time",
              "name": null,
              "show": true,
              "values": []
            },
            "yaxes": [
              {
                "format": "percentunit",
                "label": null,
                "logBase": 1,
                "max": "1",
                "min": "0",
                "show": true
              },
              {
                "format": "short",
                "label": null,
                "logBase": 1,
                "max": null,
                "min": null,
                "show": true
              }
            ],
            "yaxis": {
              "align": false,
              "alignLevel": null
            }
          }
        ],
        "refresh": false,
        "schemaVersion": 22,
        "style": "dark",
        "tags": [],
        "templating": {
          "list": [
            {
              "current": {
                "selected": true,
                "text": "Prometheus",
                "value": "Prometheus"
              },
              "hide": 0,
              "includeAll": false,
              "multi": false,
              "name": "datasource",
              "options": [],
              "query": "prometheus",
              "queryValue": "",
              "refresh": 1,
              "regex": "",
              "skipUrlSync": false,
              "type": "datasource"
            },
            {
              "allValue": null,
              "current": {},
              "datasource": "$datasource",
              "definition": "label_values(DCGM_FI_DEV_GPU_TEMP, instance)",
              "hide": 0,
              "includeAll": true,
              "index": -1,
              "label": null,
              "multi": true,
              "name": "instance",
              "options": [],
              "query": "label_values(DCGM_FI_DEV_GPU_TEMP, instance)",
              "refresh": 1,
              "regex": "",
              "skipUrlSync": false,
              "sort": 1,
              "tagValuesQuery": "",
              "tags": [],
              "tagsQuery": "",
              "type": "query",
              "useTags": false
            },
            {
              "allValue": null,
              "current": {},
              "datasource": "$datasource",
              "definition": "label_values(DCGM_FI_DEV_GPU_TEMP, gpu)",
              "hide": 0,
              "includeAll": true,
              "index": -1,
              "label": null,
              "multi": true,
              "name": "gpu",
              "options": [],
              "query": "label_values(DCGM_FI_DEV_GPU_TEMP, gpu)",
              "refresh": 1,
              "regex": "",
              "skipUrlSync": false,
              "sort": 1,
              "tagValuesQuery": "",
              "tags": [],
              "tagsQuery": "",
              "type": "query",
              "useTags": false
            }
          ]
        },
        "time": {
          "from": "now-15m",
          "to": "now"
        },
        "timepicker": {
          "refresh_intervals": [
            "5s",
            "10s",
            "30s",
            "1m",
            "5m",
            "15m",
            "30m",
            "1h",
            "2h",
            "1d"
          ]
        },
        "timezone": "",
        "title": "NVIDIA DCGM Exporter Dashboard",
        "uid": "Oxed_c6Wz",
        "variables": {
          "list": []
        },
        "version": 1
      }
  ```
</details>

With that ConfigMap created you should be able to navigate to the Observe > Dashboards section of the OpenShift dashboard and select the "NVIDIA DCGM Exporter Dashboard" from the Dashboard drop down.