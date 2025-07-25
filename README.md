# Exascaler-csi-file-driver

Releases can be found here - https://github.com/DDNStorage/exa-csi-driver/releases

## Compatibility matrix
|CSI driver version|EXAScaler client version|EXA Scaler server version|
|--- |---|---|
|>=v2.3.0|>=2.14.0-ddn182|>=6.3.2|

## Feature List
|Feature|Feature Status|CSI Driver Version|CSI Spec Version|Kubernetes Version|Openshift Version|
|--- |--- |--- |--- |--- |--- |
|Static Provisioning|GA|>= 1.0.0|>= 1.0.0|>=1.18|>=4.13|
|Dynamic Provisioning|GA|>= 1.0.0|>= 1.0.0|>=1.18|>=4.13|
|RW mode|GA|>= 1.0.0|>= 1.0.0|>=1.18|>=4.13|
|RO mode|GA|>= 1.0.0|>= 1.0.0|>=1.18|>=4.13|
|Expand volume|GA|>= 1.0.0|>= 1.1.0|>=1.18|>=4.13|
|StorageClass Secrets|GA|>= 1.0.0|>=1.0.0|>=1.18|>=4.13|
|Mount options|GA|>= 1.0.0|>= 1.0.0|>=1.18|>=4.13|
|Topology|GA|>= 2.0.0|>= 1.0.0|>=1.17|>=4.13|
|Snapshots|GA|>= 2.2.6|>= 1.0.0|>=1.17|>=4.13|
|Exascaler Hot Nodes|GA|>= 2.3.0|>= 1.0.0|>=1.18| Not supported yet|

## Access Modes support
|Access mode| Supported in version|
|--- |--- |
|ReadWriteOnce| >=1.0.0 |
|ReadOnlyMany| >=2.2.3 |
|ReadWriteMany| >=1.0.0 |
|ReadWriteOncePod| >=2.2.3 |

## OpenShift Certification
|OpenShift Version| CSI driver Version| EXA Version|
|---|---|---|
|v4.13|>=v2.2.3|>=v6.3.0|
|v4.14|>=v2.2.4|>=v6.3.0|
|v4.15|>=v2.2.4|>=v6.3.0|

## OpenShift
### Prerequisites
Internal OpenShift image registry needs to be patched to allow building lustre modules with KMM.
```bash
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
oc patch configs.imageregistry.operator.openshift.io/cluster --type merge -p '{"spec":{"defaultRoute":true}}'
```

### Building lustre rpms
You will need a vm with the kernel version matching that of the OpenShift nodes. To check on nodes:
```bash
oc get nodes
oc debug node/c1-pk6k4-worker-0-2j6w8
uname -r
```

On the builder vm, install the matching kernel and log in to it, example for Rhel 9.2:
```bash
# login to subscription-manager
subscription-manager register --username <username> --password <password> --auto-attach
# list available kernels
yum --showduplicates list available kernel
yum install kernel-<version>-<revision>.<arch>  # e.g.: yum install kernel-5.14.0-284.25.1.el9_2.x86_64
grubby --info=ALL | grep title
title="Red Hat Enterprise Linux (5.14.0-284.11.1.el9_2.x86_64) 9.2 (Plow)"                <---- 0
title="Red Hat Enterprise Linux (5.14.0-284.25.1.el9_2.x86_64) 9.2 (Plow)"                <---- 1

grub2-set-default 1
reboot
```

Copy EXAScaler client tar from the EXAScaler server:
```bash
scp root@<exa-server-ip>:/scratch/EXAScaler-<version>/exa-client-<version>.tar.gz .
tar -xvf exa-client-<version>.tar.gz
cd exa-client
./exa_client_deploy.py -i
```

This will build the rpms and install the client.
Upload the rpms to any repository available from the cluster and change deploy/openshift/lustre-module/lustre-dockerfile-configmap.yaml lines 12-13 accordingly.
Make sure that `kmod-lustre-client-*.rpm`, `lustre-client-*.rpm` and `lustre-client-devel-*.rpm` packages are present.
  ```
  RUN git clone https://github.com/Qeas/rpms.git # change this to your repo with matching rpms
  RUN yum -y install rpms/*.rpm
  ```

### Loading lustre modules in OpenShift

Before loading the lustre modules, make sure to install OpenShift Kernel Module Management (KMM) via OpenShift console.

```bash
oc create -n openshift-kmm -f deploy/openshift/lustre-module/lustre-dockerfile-configmap.yaml
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lnet-mod.yaml
```
Wait for the builder pod (e.g. `lnet-build-5f265-build`) to finish. After builder finishes,
you should have `lnet-8b72w-6fjwh` running on each worker node.
```bash
# run ko2iblnd-mod if you are using Infiniband network
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/ko2iblnd-mod.yaml
```

Make changes to `deploy/openshift/lustre-module/lnet-configuration-ds.yaml` line 38 according to the cluster's network
```
lnetctl net add --net tcp --if br-ex # change interface according to your cluster
```
Configure lnet and install lustre
```
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lnet-configuration-ds.yaml
oc apply -n openshift-kmm -f deploy/openshift/lustre-module/lustre-mod.yaml
```

### Installing the driver

Make sure that `openshift: true` in `deploy/openshift/exascaler-csi-file-driver-config.yaml`.
Create a secret from the config file and apply the driver yaml.

```bash
oc create -n openshift-kmm secret generic exascaler-csi-file-driver-config --from-file=deploy/openshift/exascaler-csi-file-driver-config.yaml
oc apply -n openshift-kmm -f deploy/openshift/exascaler-csi-file-driver.yaml
```

### Uninstall

```bash
oc delete -n openshift-kmm secret exascaler-csi-file-driver-config
oc delete -n openshift-kmm -f deploy/openshift/exascaler-csi-file-driver.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/lustre-mod.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/lnet-configuration-ds.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/ko2iblnd-mod.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/lnet-mod.yaml
oc delete -n openshift-kmm -f deploy/openshift/lustre-module/lustre-dockerfile-configmap.yaml
oc get images | grep lustre-client-moduleloader | awk '{print $1}' | xargs oc delete image
```


### Snapshots
To use CSI snapshots, the snapshot CRDs along with the csi-snapshotter must be installed.
```bash
oc apply -f deploy/openshift/snapshots/
```

After that the snapshot class for EXA CSI must be created
Snapshot parameters can be passed through snapshot class as can be seen is `examples/snapshot-class.yaml`
List of available snapshot parameters:

| Name           | Description                                                       | Example                              |
|----------------|-------------------------------------------------------------------|--------------------------------------|
| `snapshotFolder` | [Optional] Folder on ExaScaler filesystem where the snapshots will be created. | `csi-snapshots` |
| `snapshotUtility` | [Optional] Either `tar` or `dtar`. Default is `tar`| `dtar` |
| `snapshotMd5Verify` | [Optional] Defines whether the driver should do md5sum check on the snapshot. Ensures that the snapshot is not corrupt but reduces performance. Default is `false` | `true` |
| `exaFS` | Same parameter as in storage class. If the original volume was created using storage class parameter for `exaFS` this MUST match the value of storage class. | `10.3.3.200@tcp:/csi-fs` |
| `mountPoint` | Same parameter as in storage class. If the original volume was created using storage class parameter for `mountPoint` this MUST match the value of storage class. | `/exa-csi-mnt` |

```bash
oc apply -f examples/snapshot-class.yaml
```

Now a snapshot can be created, for example
```bash
oc apply -f examples/snapshot-from-dynamic.yaml
```

We can create a volume using the snapshot
```bash
oc apply -f examples/nginx-from-snapshot.yaml
```

Exascaler CSI driver supports 2 snapshot modes: `tar` or `dtar`.
Default mode is `tar`. Dtar is much faster.
To enable `dtar` set `snapshotUtility: dtar` in config.


## Kubernetes
### Requirements

- Required API server and kubelet feature gates for k8s version < 1.16 (skip this step for k8s >= 1.16)
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enabling-features)):
  ```
  --feature-gates=ExpandInUsePersistentVolumes=true,ExpandCSIVolumes=true,ExpandPersistentVolumes=true
  ```
- Mount propagation must be enabled, the Docker daemon for the cluster must allow shared mounts
  ([instructions](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation))

- MpiFileUtils dtar must be installed on all kubernetes nodes in order to use `dtar` as snapshot utility. Not required if `tar` is used for snapshots (default).

### Prerequisites
EXAScaler client must be installed and configured on all kubernetes nodes. Please refer to EXAScaler Installation and Administration Guide.

### Installation
Clone or untar driver (depending on where you get the driver)
```bash
git clone -b <driver version> https://github.com/DDNStorage/exa-csi-driver.git /opt/exascaler-csi-file-driver
```
e.g:-
```bash
git clone -b 2.2.4 https://github.com/DDNStorage/exa-csi-driver.git /opt/exascaler-csi-file-driver
```
or
```bash
rpm -Uvh exa-csi-driver-1.0-1.el7.x86_64.rpm
```

### Using helm chart

Pull latest helm chart configuration before installing or upgrading. 

#### Install
- Make changes to `deploy/helm-chart/values.yaml` according to your Kubernetes and EXAScaler clusters environment.

- If metrics exporter is required, make changes to `deploy/helm-chart/values.yaml` according to your environment. refer to [metrics exporter configuration](#metrics-exporter-configuration)

- Run `helm install -n ${namespace} exascaler-csi-file-driver deploy/helm-chart/`

#### Uninstall
`helm uninstall -n ${namespace} exascaler-csi-file-driver`

#### Upgrade

- Make any necessary changes to the chart, for example new driver version: `tag: "v2.3.4"` in `deploy/helm-chart/values.yaml`.
- If metrics exporter is required, make changes to `deploy/helm-chart/values.yaml` according to your environment. refer to [metrics exporter configuration](#metrics-exporter-configuration)
- Run `helm upgrade -n ${namespace} exascaler-csi-file-driver deploy/helm-chart/`


### Metrics exporter configuration.

List of exported metrics:
- exa_csi_pvc_capacity_bytes - Total capacity in bytes of PVs with provisioner exa.csi.ddn.com
- exa_csi_pvc_used_bytes - Used bytes of PVs with provisioner exa.csi.ddn.com
- exa_csi_pvc_available_bytes - Available bytes of PVs with provisioner exa.csi.ddn.com
- exa_csi_pvc_pod_count - Number of pods using a PVC with provisioner exa.csi.ddn.com

Each of those metrics reports with 3 labels: "pvc", "storage_class", "exported_namespace".
These labels can be used to group the metrics by kubernetes storage class and namespace as shown below in [Add Prometheus Rules for StorageClass and Namespace Aggregation](#add-prometheus-rules-for-storageclass-and-namespace-aggregation)

```yaml
metrics:
  enabled: true      # Enable metrics exporter 
  exporter:
    repository: quay.io/ddn/exa-csi-metrics-exporter
    tag: master # use latest version same as driver version e.g - tag: "2.3.4"
    pullPolicy: Always
    containerPort: "9200" # Metrics Exporter port
    servicePort: "9200" # This port will be used to create service for exporter
    namespace: default # Namespace where metrics exporter will be deployed
    logLevel: info     # Log level. Debug is not recommended for production, default is info.
    collectTimeout: 10 # Metrics refresh timeout in seconds. Default is 10.
  serviceMonitor:
    enabled: false # set to true if using prometheus operator
    interval: 30s
    namespace: monitoring # Namespace where prometheus operator is deployed
  prometheus:                                                 
    releaseLabel: prometheus      # release label for prometheus operator                 
    createClusterRole: false   # set to false if using existing ClusterRole
    createClustrerRoleName: exa-prometheus-cluster-role  # Name of ClusterRole to create, this name will be used to bind cluster role to prometheus service account
    clusterRoleName: cluster-admin  # Name of ClusterRole to bind to prometheus service account if createClusterRole is set to false otherwise `createClustrerRoleName` will be used
    serviceAccountName: prometheus-kube-prometheus-prometheus # Service account name of prometheus

```

## Prometheus Server Integration

This section describes how to:
- Add the EXAScaler CSI metrics exporter to Prometheus' scrape configuration.
- Apply PrometheusRule definitions for metrics aggregation by StorageClass and Namespace.

### 1. Add EXAScaler CSI Metrics Exporter to Prometheus

#### Step 1.1 – Create a Secret with Scrape Config

Get the hostPort number of the EXAScaler CSI metrics exporter:
```bash
kubectl get daemonset exa-csi-metrics-exporter -o jsonpath='{.spec.template.spec.containers[*].ports[*].hostPort}{"\n"}'

32666
```

Create a Kubernetes secret that holds your additional Prometheus scrape configuration:
```bash
PORT=32666
cat <<EOF | envsubst > additional-scrape-configs.yaml
- job_name: 'exa-csi-metrics-exporter-remote'
  static_configs:
    - targets:
        - 10.20.30.1:$PORT
        - 10.20.30.2:$PORT
        - 10.20.30.3:$PORT
EOF
kubectl create secret generic additional-scrape-configs \
  --from-file=additional-scrape-configs.yaml -n monitoring
```
Note: Replace the targets value with the actual IP and port of your EXAScaler CSI metrics exporter.

#### Step 1.2 – Patch the Prometheus Custom Resource

Edit the Prometheus CR (custom resource) and add the additionalScrapeConfigs reference:
```bash
kubectl edit prometheus prometheus-kube-prometheus-prometheus -n monitoring
```
Add the following under spec:
```yaml
  additionalScrapeConfigs:
    name: additional-scrape-configs
    key: additional-scrape-configs.yaml
```

#### Step 1.3 – Restart Prometheus

After editing the CR, restart Prometheus so it reloads the configuration:
```bash
kubectl delete pod -l app.kubernetes.io/name=prometheus -n monitoring
```

### 2. Add Prometheus Rules for StorageClass and Namespace Aggregation

Apply a PrometheusRule resource to aggregate PVC metrics by StorageClass and Namespace:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: exa-csi-rules
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: exa-storageclass-rules
      interval: 15s
      rules:
        - record: exa_csi_sc_capacity_bytes
          expr: sum(exa_csi_pvc_capacity_bytes) by (storage_class)
        - record: exa_csi_sc_used_bytes
          expr: sum(exa_csi_pvc_used_bytes) by (storage_class)
        - record: exa_csi_sc_available_bytes
          expr: sum(exa_csi_pvc_available_bytes) by (storage_class)
        - record: exa_csi_sc_pvc_count
          expr: count(exa_csi_pvc_capacity_bytes) by (storage_class)

    - name: exa-namespace-rules
      interval: 15s
      rules:
        - record: exa_csi_namespace_capacity_bytes
          expr: sum(exa_csi_pvc_capacity_bytes) by (exported_namespace)
        - record: exa_csi_namespace_used_bytes
          expr: sum(exa_csi_pvc_used_bytes) by (exported_namespace)
        - record: exa_csi_namespace_available_bytes
          expr: sum(exa_csi_pvc_available_bytes) by (exported_namespace)
        - record: exa_csi_namespace_pvc_count
          expr: count(exa_csi_pvc_capacity_bytes) by (exported_namespace)
```
Apply the rule:
```bash
kubectl apply -f exa-csi-rules.yaml
```
Ensure that Prometheus is watching the monitoring namespace for PrometheusRule CRDs.


### Using docker load and kubectl commands
   ```bash
   docker load -i /opt/exascaler-csi-file-driver/bin/exascaler-csi-file-driver.tar
   ```
#### Verify version
kubectl get deploy/exascaler-csi-controller -o jsonpath="{..image}"

2. Copy /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver-config.yaml to /etc/exascaler-csi-file-driver-v1.0/exascaler-csi-file-driver-config.yaml
   ```
   cp /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver-config.yaml /etc/exascaler-csi-file-driver-v1.0/exascaler-csi-file-driver-config.yaml 
   ```
Edit `/etc/exascaler-csi-file-driver-v1.0/exascaler-csi-file-driver-config.yaml` file. Driver configuration example:
   ```yaml
   exascaler_map:
     exa1:
       mountPoint: /exaFS                                          # mountpoint on the host where the exaFS will be mounted
       exaFS: 192.168.88.114@tcp2:192.168.98.114@tcp2:/testfs                            # default path to exa filesystem where the PVCs will be stored
       managementIp: 10.204.86.114@tcp                           # network for management operations, such as create/delete volume
       zone: zone-1

     exa2:
       mountPoint: /exaFS-zone-2                                          # mountpoint on the host where the exaFS will be mounted
       exaFS: 192.168.78.112@tcp2:/testfs/zone-2                            # default path to exa filesystem where the PVCs will be stored
       managementIp: 10.204.86.114@tcp                           # network for management operations, such as create/delete volume
       zone: zone-2

     exa3:
       mountPoint: /exaFS-zone-3                                          # mountpoint on the host where the exaFS will be mounted
       exaFS: 192.168.98.113@tcp2:192.168.88.113@tcp2:/testfs/zone-3                            # default path to exa filesystem where the PVCs will be stored
       managementIp: 10.204.86.114@tcp                           # network for management operations, such as create/delete volume
       zone: zone-3

   debug: true
   ```

3. Create Kubernetes secret from the file:
   ```bash
   kubectl create secret generic exascaler-csi-file-driver-config --from-file=/etc/exascaler-csi-file-driver-v1.0/exascaler-csi-file-driver-config.yaml
   ```

4. Register driver to Kubernetes:
   ```bash
   kubectl apply -f /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver.yaml
   ```

## Usage

### Dynamically provisioned volumes

For dynamic volume provisioning, the administrator needs to set up a _StorageClass_ in the PV yaml (/opt/exascaler-csi-file-driver/examples/exa-dynamic-nginx.yaml for this example) pointing to the driver.
For dynamically provisioned volumes, Kubernetes generates volume name automatically (for example `pvc-ns-cfc67950-fe3c-11e8-a3ca-005056b857f8-projectId-1001`).

Basic storage class example, uses config to access Exascaler
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-file-driver-sc-nginx-dynamic
provisioner: exa.csi.ddn.com
configName: exa1
```

The following example shows how to use storage class to override config values
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-file-driver-sc-nginx-dynamic
provisioner: exa.csi.ddn.com
allowedTopologies:
- matchLabelExpressions:
  - key: topology.exa.csi.ddn.com/zone
    values:
    - zone-1
mountOptions:                        # list of options for `mount -o ...` command
#  - noatime                         #
parameters:
  exaMountUid: "1001"      # Uid which will be used to access the volume in pod. Should be synced between EXA server and clients.
  exaMountGid: "1002"      # Gid which will be used to access the volume in pod. Should be synced between EXA server and clients.
  bindMount: "true"       # Determines, whether volume will bind mounted or as a separate lustre mount.
  exaFS: "10.204.86.114@tcp:/testfs"   # Overrides exaFS value from config. Use this to support multiple EXA filesystems.
  mountPoint: /exaFS       # Overrides mountPoint value from config. Use this to support multiple EXA filesystems.
  mountOptions: ro,noflock
  minProjectId: 10001      # project id range for this storage class
  maxProjectId: 20000
```

#### Example

Run Nginx pod with dynamically provisioned volume:

```bash
kubectl apply -f /opt/exascaler-csi-file-driver/examples/exa-dynamic-nginx.yaml

# to delete this pod:
kubectl delete -f /opt/exascaler-csi-file-driver/examples/exa-dynamic-nginx.yaml
```

### Static (pre-provisioned) volumes

The driver can use already existing Exasaler filesystem,
in this case, _StorageClass_, _PersistentVolume_ and _PersistentVolumeClaim_ should be configured.

Quota can be manually assigned for static volumes. First we need to associate a project with the volume folder
```bash
lfs project -p 1000001 -s /mountPoint-csi/nginx-persistent
```

Then a quota can be set for that project
```bash
lfs setquota -p 1000001 -B 2G /mountPoint-csi/
```

#### _StorageClass_ configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-driver-sc-nginx-persistent
provisioner:  exa.csi.ddn.com
allowedTopologies:
- matchLabelExpressions:
  - key: topology.exa.csi.ddn.com/zone
    values:
    - zone-1
mountOptions:                         # list of options for `mount -o ...` command
#  - noatime                         #
```

#### _PersistentVolume_ configuration

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: exascaler-csi-driver-pv-nginx-persistent
  labels:
    name: exascaler-csi-driver-pv-nginx-persistent
spec:
  storageClassName: exascaler-csi-driver-sc-nginx-persistent
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi
  csi:
    driver: exa.csi.ddn.com
    volumeHandle: exa1:10.3.3.200@tcp;/exaFS:/mountPoint-csi:/nginx-persistent
    volumeAttributes:         # volumeAttributes are the alternative of storageClass params for static (precreated) volumes.
      exaMountUid: "1001"     # Uid which will be used to access the volume in pod.
      #mountOptions: ro, flock # list of options for `mount` command
      projectId: "1000001"
```

### Topology configuration
In order to configure CSI driver with kubernetes topology, use the `zone` parameter in driver config or storageClass parameters. Example config file with zones:
  ```bash

  exascaler_map:
    exa1:
      exaFS: 10.3.196.24@tcp:/csi
      mountPoint: /exaFS
      zone: us-west
    exa2:
      exaFS: 10.3.196.24@tcp:/csi-2
      mountPoint: /exaFS-zone2
      zone: us-east
  ```

This will assign volumes to be created on Exascaler cluster that correspond with the zones requested by allowedTopologies values.

For topology-aware scheduling:

Each node must have the `topology.exa.csi.ddn.com/zone` label with the zone configuration.

Label node with topology zone:
```bash
kubectl label node <node-name> topology.exa.csi.ddn.com/zone=us-west
```
For removing label:
```bash
kubectl label node <node-name> topology.exa.csi.ddn.com/zone-
```

Important: If topology labels are added after driver installation, the node driver must be restarted on affected nodes.

Restart node driver pod on specific node:
```bash
kubectl delete pod -l app=exascaler-csi-node -n <namespace> --field-selector spec.nodeName=<node-name>
```

or restart all node driver pods:
```bash
kubectl rollout restart daemonset exascaler-csi-node -n <namespace>
```

For using `volumeBindingMode: WaitForFirstConsumer`, the topology must be configured.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-file-driver-sc-nginx-dynamic
provisioner: exa.csi.ddn.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  configName: exa1
```

### CSI Parameters
If a parameter is available for both config and storage class, storage class parameter will override config values

|Config|Storage class| Description                                                       | Example                              |
|--------|------------|------------------------------------------------------------------|--------------------------------------|
| `exaFS` | `exaFS` | [required] Full path to EXAScaler filesystem | `10.3.3.200@tcp:/csi-fs` |
| `mountPoint`  |`mountPoint` | [required] Mountpoint on Kubernetes host where the exaFS will be mounted | `/exa-csi-mnt` |
| - | `driver`    [required]  |  Installed driver name " exa.csi.ddn.com"        | `exa.csi.ddn.com` |
| - | `volumeHandle` | [required for static volumes] The format is &lt;configName&gt;:&lt;NID&gt;;&lt;Exascaler filesystem path&gt;:&lt;mountPoint&gt;:&lt;volumeHandle&gt;. **Note:** The NID and Exascaler filesystem path are separated by a semicolon ( ; ), all other fields are delimited by a colon ( : ). | `exa1:10.3.3.200@tcp;/exaFS:/mountPoint-csi:/nginx-persistent`      |
| - | `exaMountUid`  | Uid which will be used to access the volume from the pod. | `1015` |
| - | `exaMountGid`  | Gid which will be used to access the volume from the pod. | `1015` |
| - | `projectId`    | Points to EXA project id to be used to set volume quota. Automatically generated by the driver if not provided. | `100001` |
| `managementIp` | `managementIp` | Should be used if there is a separate network configured for management operations, such as create/delete volumes. This network should have access to all Exa filesystems in a isolated zones environment | `192.168.10.20@tcp2` |
| `bindMount` | `bindMount`    | Determines, whether volume will bind mounted or as a separate lustre mount. Default is `true` | `true` |
| `defaultMountOptions` | `mountOptions` | Options that will be passed to mount command (-o <opt1,opt2,opt3>)  | `ro,flock` |
| - | `minProjectId` | Minimum project ID number for automatic generation. Only used when projectId is not provided. | 10000 |
| - | `maxProjectId` | Maximum project ID number for automatic generation. Only used when projectId is not provided. | 4294967295 |
| - | `generateProjectIdRetries` | Maximum retry count for generating random project ID. Only used when projectId is not provided. | `5` |
| `zone` | `zone`        | Topology zone to control where the volume should be created. Should match topology.exa.csi.ddn.com/zone label on node(s). | `us-west` |
| `v1xCompatible` | - | [Optional] Only used when upgrading the driver from v1.x.x to v2.x.x. Provides compatibility for volumes that were created beore the upgrade. Set it to `true` to point to the Exa cluster that was configured before the upgrade | `false` |
|`tempMountPoint` | `tempMountPoint` | [Optional] Used when `exaFS` points to a subdirectory that does not exist on Exascaler and will be automatically created by the driver. This parameter sets the directory where Exascaler filesystem will be temporarily mounted to create the subdirectory. | `/tmp/exafs-mnt` |
| `volumeDirPermissions` | `volumeDirPermissions` | [Optional] Defines file permissions for mounted volumes. | `0777` |
| `hotNodes` | `hotNodes` | Determines whether `HotNodes` feature should be used. This feature can only be used by the driver when Hot Nodes (PCC) service is disabled and not used manually on the kubernetes workers. | `false` |
| `pccCache` | `pccCache` | Directory for cached files of the file system. Note that lpcc does not recognize directories with a
trailing slash (“/” at the end). | `/csi-pcc` |
| `pccAutocache` | `pccAutocache` | Condition for automatic file attachment (caching) | `projid={500}` |
| `pccPurgeHighUsage` | `pccPurgeHighUsage` | If the disk usage of cache device is higher than high_usage, start detaching the files. Defaults to 90 (90% disk/inode usage). | `90` |
| `pccPurgeLowUsage` | `pccPurgeLowUsage` | If the disk usage of cache device is lower than low_usage, stop detaching the files. Defaults to 75 (75% disk/inode usage). | `70` |
| `pccPurgeScanThreads` | `pccPurgeScanThreads` | Threads to use for scanning cache device in parallel. Defaults to 1. | `1` |
| `pccPurgeInterval` | `pccPurgeInterval` | Interval for lpcc_purge to check cache device usage, in seconds. Defaults to 30. | `30` |
| `pccPurgeLogLevel` | `pccPurgeLogLevel` | Log level for lpcc_purge: either “fatal”, “error”, “warn”, “normal”, “info” (default), or “debug”. | `info` |
| `pccPurgeForceScanInterval` | `pccPurgeForceScanInterval` | Scan PCC backends forcefully after this number of seconds to refresh statistic data. | `30` |
| - | `compression` | Algorithm ["lz4", "gzip", "lzo"] to use for data compression. default is "false" | `false` | 
| - | `configName` | Config entry name to use from the config map | `exa1` |

#### _PersistentVolumeClaim_ (pointing to created _PersistentVolume_)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: exascaler-csi-driver-pvc-nginx-persistent
spec:
  storageClassName: exascaler-csi-driver-cs-nginx-persistent
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      # to create 1-1 relationship for pod - persistent volume use unique labels
      name: exascaler-csi-file-driver-pv-nginx-persistent
```

#### Example

Run nginx server using PersistentVolume.

**Note:** Pre-configured filesystem should exist on the EXAScaler:
`/exaFS/nginx-persistent`.

```bash
kubectl apply -f /opt/exascaler-csi-file-driver/examples/nginx-persistent-volume.yaml

# to delete this pod:
kubectl delete -f /opt/exascaler-csi-file-driver/examples/nginx-persistent-volume.yaml
```

### Snapshots
To use CSI snapshots, the snapshot CRDs along with the csi-snapshotter must be installed.
```bash
kubectl apply -f deploy/kubernetes/snapshots/
```

After that the snapshot class for EXA CSI must be created
Snapshot parameters can be passed through snapshot class as can be seen is `examples/snapshot-class.yaml`
List of available snapshot parameters:

| Name           | Description                                                       | Example                              |
|----------------|-------------------------------------------------------------------|--------------------------------------|
| `snapshotFolder` | [Optional] Folder on ExaScaler filesystem where the snapshots will be created. | `csi-snapshots` |
| `snapshotUtility` | [Optional] Either `tar` or `dtar`.  `dtar` is faster but requires `dtar` to be installed on all k8s nodes. Default is `tar`| `dtar` |
| `dtarPath` | [Optional] If `snapshotUtility` is `dtar` points to where the `dtar` utility is installed | `/opt/ddn/mpifileutils/bin/dtar` |
| `snapshotMd5Verify` | [Optional] Defines whether the driver should do md5sum check on the snapshot. Ensures that the snapshot is not corrupt but reduces performance. Default is `false` | `true` |
| `exaFS` | Same parameter as in storage class. If the original volume was created using storage class parameter for `exaFS` this MUST match the value of storage class. | `10.3.3.200@tcp:/csi-fs` |
| `mountPoint` | Same parameter as in storage class. If the original volume was created using storage class parameter for `mountPoint` this MUST match the value of storage class. | `/exa-csi-mnt` |

```bash
kubectl apply -f examples/snapshot-class.yaml
```

Now a snapshot can be created, for example
```bash
kubectl apply -f examples/snapshot-from-dynamic.yaml
```

We can create a volume using the snapshot
```bash
kubectl apply -f examples/nginx-from-snapshot.yaml
```

Exascaler CSI driver supports 2 snapshot modes: `tar` or `dtar`.
Default mode is `tar`.
To enable `dtar` set `snapshotUtility: dtar` in config.
Dtar is much faster but requires mpifileutils `dtar` installed on all the nodes as a prerequisite.
Installing `dtar` might differ depending on your OS. Here is an example for Ubuntu 22

```bash
# Install dependencies mpifileutils
cd
mkdir -p /opt/ddn/mpifileutils
cd /opt/ddn/mpifileutils
sudo apt-get install -y cmake libarchive-dev libbz2-dev libcap-dev libssl-dev openmpi-bin libopenmpi-dev libattr1-dev

export INSTALL_DIR=/opt/ddn/mpifileutils

git clone https://github.com/LLNL/lwgrp.git
cd lwgrp
./autogen.sh
./configure --prefix=$INSTALL_DIR
make -j 16
make install
cd ..

git clone https://github.com/LLNL/dtcmp.git
cd dtcmp
./autogen.sh
./configure --prefix=$INSTALL_DIR --with-lwgrp=$INSTALL_DIR
make -j 16
make install
cd ..

git clone https://github.com/hpc/libcircle.git
cd libcircle
sh ./autogen.sh
./configure --prefix=$INSTALL_DIR
make -j 16
make install
cd ..

git clone https://github.com/hpc/mpifileutils.git mpifileutils-clone
cd mpifileutils-clone
cmake ./ -DWITH_DTCMP_PREFIX=$INSTALL_DIR -DWITH_LibCircle_PREFIX=$INSTALL_DIR -DCMAKE_INSTALL_PREFIX=$INSTALL_DIR -DENABLE_LUSTRE=ON -DENABLE_XATTRS=ON
make -j 16
make install
cd ..
```

If you use a different `INSTALL_DIR` path, pass it in config using `DtarPath` parameter.

### Encryption
- Limitations:
If you are planning to use volumes from more than one Exascaler Filesystem on a single worker node, then the encryption does not work in the current release.

To use encrypted volumes, first create a secret with a passphrase.
```bash
kubectl create secret generic exa-csi-sc1 --from-literal=passphrase=pass1
```

Then, enable encryption in storage class parameters and pass the secret:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-file-driver-sc-1
provisioner: exa.csi.ddn.com
allowVolumeExpansion: true
parameters:
  csi.storage.k8s.io/provisioner-secret-name: exa-csi-sc1
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/node-publish-secret-name: exa-csi-sc1
  csi.storage.k8s.io/node-publish-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: exa-csi-sc1
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  encryption: "true"
```

For static provisioning, encryption should be enabled on the volume before using it in kubernetes CSI.
Make sure to use a protector (existing or create new one) with the same passphrase as in the secret.
```yaml
fscrypt setup /exa-mount

echo "pass1" | fscrypt metadata create protector /exa-mount --source=custom_passphrase --name=persistent-protector --quiet

fscrypt status /exa-mount


protectorid=$(fscrypt status /exa-mount | awk '/persistent-protector/ {print $1}')
echo "pass1" | fscrypt encrypt /exa-mount/static-enc --protector=/exa-mount:$protectorid --quiet

```

### Configuring the driver to use with ExaScaler nodemap.
If you have a nodemap where some nodes mount subfolder instead of root filesystem, you will need
to have at least 1 dedicated node able to mount the ExaScaler root fs to run the controller driver.
Use `managementExaFs` secret to pass the full path to the mapped subdirectory.

For example, we have the following filesystem:
`192.168.5.1@tcp,192.168.6.1@tcp:/fs` with 2 subfolders `tenant1` and `tenant2`.
Create 2 secrets (if using with encryption, bot `passphrase` and `managementExaFs` should be in the secret)

```bash
kubectl create secret generic exa-csi-sc1 --from-literal=passphrase=pass1 --from-literal=managementExaFs=192.168.5.1@tcp,192.168.6.1@tcp:/fs/tenant1
```

```bash
kubectl create secret generic exa-csi-sc2 --from-literal=passphrase=pass2 --from-literal=managementExaFs=192.168.5.1@tcp,192.168.6.1@tcp:/fs/tenant2
```

And 2 storage classes:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-file-driver-sc-1
provisioner: exa.csi.ddn.com
allowVolumeExpansion: true
parameters:
  csi.storage.k8s.io/provisioner-secret-name: exa-csi-sc1
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/node-publish-secret-name: exa-csi-sc1
  csi.storage.k8s.io/node-publish-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: exa-csi-sc1
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  encryption: "true"
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: exascaler-csi-file-driver-sc-2
provisioner: exa.csi.ddn.com
allowVolumeExpansion: true
parameters:
  csi.storage.k8s.io/provisioner-secret-name: exa-csi-sc2
  csi.storage.k8s.io/provisioner-secret-namespace: default
  csi.storage.k8s.io/node-publish-secret-name: exa-csi-sc2
  csi.storage.k8s.io/node-publish-secret-namespace: default
  csi.storage.k8s.io/controller-expand-secret-name: exa-csi-sc1
  csi.storage.k8s.io/controller-expand-secret-namespace: default
  encryption: "true"
```

#### Scheduling controller driver on management node
We need to run the controller driver on a dedicated node that has permissions to mount
`192.168.5.1@tcp,192.168.6.1@tcp:/fs`.
And we need to ensure that applications pods are not scheduled on this node, we will use `NoSchedule` taint to 
ensure that. *Note:* There are other ways in kubernetes to configure this, such as `nodeSelector`, `affinity` and `topologySpreadConstraints`.
```bash
kubectl taint nodes node1 management:NoSchedule
```
Add a `toleration` and a `nodeName` to our driver `Deployment`.
For helm chart deployment: https://github.com/DDNStorage/exa-csi-driver/blob/master/deploy/helm-chart/templates/controller-driver.yaml#L203
For generic kubernetes yaml files: https://github.com/DDNStorage/exa-csi-driver/blob/master/deploy/kubernetes/exascaler-csi-file-driver.yaml#L216

```yaml
spec:
  # serviceName: exascaler-csi-controller-service
  selector:
    matchLabels:
      app: exascaler-csi-controller # has to match .spec.template.metadata.labels
  template:
    metadata:
      labels:
        app: exascaler-csi-controller
    spec:
      serviceAccount: exascaler-csi-controller-service-account
      priorityClassName: {{ .Values.priorityClassName }}
      tolerations:
        - key: "management"
          operator: "Exists"
          effect: "NoSchedule"
      nodeName: "node1"
```
This will ensure that controller driver is running on management node.

## Updating the driver version
To update to a new driver version, you need to follow the following steps:

1. Remove the old driver version
```bash
kubectl delete -f /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver.yaml
kubectl delete secrets exascaler-csi-file-driver-config
rpm -evh exa-csi-driver
```
2. Download the new driver version (git clone or new ISO)
3. Copy and edit config file
  ```bash
  cp /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver-config.yaml /etc/exascaler-csi-file-driver-v1.1/exascaler-csi-file-driver-config.yaml
  ```

  If you are upgrading from v1.x.x to v2.x.x, config file structure will change to a map of Exascaler clusters instead of a flat structure. To support old volume that were created using v1.x.x, old config should be put in the config map with `v1xCompatible: true`. For example:

  v1.x.x config
  ```bash
  exaFS: 10.3.196.24@tcp:/csi
  mountPoint: /exaFS
  debug: true
  ```
  v2.x.x config with support of previously created volumes
  ```bash

  exascaler_map:
    exa1:
      exaFS: 10.3.196.24@tcp:10.3.199.24@tcp:/csi
      mountPoint: /exaFS
      v1xCompatible: true

    exa2:
      exaFS: 10.3.1.200@tcp:10.3.2.200@tcp:10.3.3.200@tcp:/csi-fs
      mountPoint: /mnt2

  debug: true
```

Only one of the Exascaler clusters can have `v1xCompatible: true` since old config supported only 1 cluster.

4. Update version in /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver.yaml
```
          image: exascaler-csi-file-driver:v2.2.5
```
5. Load new image
```bash
docker load -i /opt/exascaler-csi-file-driver/bin/exascaler-csi-file-driver.tar
```
6. Apply new driver
```bash
kubectl create secret generic exascaler-csi-file-driver-config --from-file=/etc/exascaler-csi-file-driver-v1.1/exascaler-csi-file-driver-config.yaml
kubectl apply -f /opt/exascaler-csi-file-driver/deploy/kubernetes/exascaler-csi-file-driver.yaml
```
7. Verify version
```bash
kubectl get deploy/exascaler-csi-controller -o jsonpath="{..image}"
```

## Troubleshooting
### Driver logs
To collect all driver related logs, you can use the `kubectl logs` command.
All in on command:
```bash
mkdir exa-csi-logs
for name in $(kubectl get pod -owide | grep exa | awk '{print $1}'); do kubectl logs $name --all-containers > exa-csi-logs/$name; done
```

To get logs from all containers of a single pod 
```bash
kubectl logs <pod_name> --all-containers
```

Logs from a single container of a pod
```bash
kubectl logs <pod_name> -c driver
```

#### Driver secret
```bash
kubectl get secret exascaler-csi-file-driver-config -o json | jq '.data | map_values(@base64d)' > exa-csi-logs/exascaler-csi-file-driver-config
```

#### PV/PVC/Pod data
```bash
kubectl get pvc > exa-csi-logs/pvcs
kubectl get pv > exa-csi-logs/pvs
kubectl get pod > exa-csi-logs/pods
```

#### Extended info about a PV/PVC/Pod:
```bash
kubectl describe pvc <pvc_name> > exa-csi-logs/<pvc_name>
kubectl describe pv <pv_name> > exa-csi-logs/<pv_name>
kubectl describe pod <pod_name> > exa-csi-logs/<pod_name>
```

#### Enabling debug logging
To enable debug logging for the driver, change debug value to `true` in either the secret for generic kubernetes installation or the `values.yaml` for helm based installation and re-deploy the driver.
To enable debug level logging for metrics, change the `logLevel` to `debug` in metrics exporter configuration. Either `https://github.com/DDNStorage/exa-csi-driver/blob/master/deploy/kubernetes/metrics/daemonset.yaml#L48` or `https://github.com/DDNStorage/exa-csi-driver/blob/master/deploy/helm-chart/values.yaml#L86` depending on how it was installed.
