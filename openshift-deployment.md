
```markdown
# OpenShift Deployment

In order to deploy the OpenShift cluster, we used JetSki, a tool that utilizes IPI (Installer-provisioned installation) which is able to deploy any size OpenShift cluster. However, there are other ways to deploy the cluster such as the Assisted Installer, which is an interactive installer wizard.

## KubeletConfig

This KubeletConfig that we applied is related to BZ#1984442. It enables us to achieve a more balanced distribution of VM pods across all nodes. This is not a must, but we would recommend it.

For more information, please refer to the Knowledge-centered support (KCS) tuning guide.

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: KubeletConfig
metadata:
  name: custom-scheduling
spec:
  machineConfigPoolSelector:
    matchLabels:
      custom-kubelet: enabled
  kubeletConfig:
    nodeStatusMaxImages: -1
```

Both custom KubeletConfigs can be enabled for worker nodes by using a label:

```bash
oc label machineconfigpool worker custom-kubelet=enable
```

Note that KubeletConfig modifications will reboot the associated nodes.

## Rate Limiter

To accommodate the potential surge in on-the-fly requests that a large-scale setup may generate during significant actions, we enhanced the rate limiting configuration. Specifically, we increased the number of queries per second (QPS) from 5 to 100, allowing the system to handle a higher request rate. Additionally, we raised the burst limit from 10 to 200, enabling the system to momentarily handle a burst of requests beyond the specified QPS threshold. For more information, please refer to the KCS tuning guide.

The updated configuration was implemented using the following command, ensuring that the system is better equipped to manage and respond to a greater volume of requests during critical operations:

```bash
oc annotate -n kubevirt-hyperconverged hco kubevirt-hyperconverged hco.kubevirt.io/tuningPolicy='{"qps":100,"burst":200}' -n openshift-cnv
```

Now we just need to enable the new values through annotation:

```bash
oc patch -n kubevirt-hyperconverged hco kubevirt-hyperconverged --type=json -p='[{"op": "add", "path": "/spec/tuningPolicy", "value": "annotation"}]' -n openshift-cnv
```

## Prometheus Config

By default, Prometheus stores metrics on ephemeral storage, which presents a limitation: any metrics data saved will be erased upon restarting or terminating the Prometheus pods. To overcome this challenge, we used a local persistent storage solution for the Prometheus database. This was achieved by allocating two SSDs, each with a 2.9TB capacity, on two designated worker nodes that host the Prometheus pods.

Taking into account the guidelines outlined in the database storage requirement documentation, we calculated that the allocation of 2.9TB proves to be adequately sufficient to comprehensively store all the metrics data amassed throughout the entirety of this release's scale test. This strategic storage approach ensures the preservation and accessibility of critical metrics data even in the face of pod restarts or terminations.

To configure local volume for Prometheus pod, we first format the two SSDs on the worker nodes with:

```bash
mkfs.ext4 /dev/nvme0n1
```

Create a mounting point:

```bash
sudo mount /dev/nvme0n1 /var/promdb
```

To make the mount persist after a reboot, we create a systemd unit file. Note the name of the file must follow the mount path separated by ‘-’.

```bash
vi var-promdb.mount
```

```ini
[Unit]
Description=Promethus Database Mount
[Mount]
What=/dev/disk/by-uuid/6b0e3351-1082-4ae7-a934-77a64cb886cd
Where=/var/promdb
Type=ext4
Options=defaults
[Install]
WantedBy=multi-user.target
```

And now we can enable those changes and make them persistent:

```bash
systemctl enable var-promdb.mount
```

Manually provision the PV using the following YAML:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: worker000-r650
spec:
  capacity:
    storage: 2975Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual-pv
  local:
    path: /var/promdb
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - worker000-r650
```

A parallel set of procedures needs to be executed for the second node, given the presence of two replicas of Prometheus pods. It's imperative to ensure consistency and redundancy in this configuration. Moreover, it's important to note that the default `persistentVolumeReclaimPolicy` is set to "delete," which necessitates an adjustment to ensure data retention. This precautionary measure involves changing the `persistentVolumeReclaimPolicy` to "retain" to safeguard against accidental deletion of the Persistent Volume (PV) or Persistent Volume Claim (PVC) and the consequential loss of valuable data.

Label the node with the corresponding node name:

```bash
oc label node worker000-r650 nodePool=promdb
oc label node worker001-r650 nodePool=promdb
```

We use the following ConfigMap YAML to bind Prometheus PVC to the PVs and attach Prometheus pods to the corresponding PVCs. Storage request size is set to 2975Gi to give Prometheus pods the full claim of that SSD.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    prometheusK8s:
      retention: 12w
      nodeSelector:
        nodePool: promdb
      volumeClaimTemplate:
        metadata:
          name: prometheusdb
        spec:
          storageClassName: manual-pv
          resources:
            requests:
              storage: 2975Gi
```

By default, Prometheus retains the data for 15 days. We extended the retention time to 12 weeks. The `nodePool` label is used to ensure that Prometheus pods will always be scheduled on the worker nodes with the Prometheus database.

Note that the above procedure can also be done using a machine config. We chose to do it in this manner in order to avoid the node reboots that occur when applying an MCP (machine configuration).

## Multus Secondary Network Config

When dealing with high-throughput applications, like managing 6000 VMs, it's advisable to use a dedicated network interface to ensure optimal performance and avoid bottlenecks. In this case, we utilized the available "ens1f1" interface as the dedicated NIC for the VMs' network operations. We achieved this by deploying the nmstate tool through the user interface (UI) for easy configuration and management, or by using the following steps:

### Create the Namespace for nmstate

```yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: openshift-nmstate
    name: openshift-nmstate
  name: openshift-nmstate
spec:
  finalizers:
  - kubernetes
```

### Add the Subscription

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  labels:
    operators.coreos.com/kubernetes-nmstate-operator.openshift-nmstate: ""
  name: kubernetes-nmstate-operator
  namespace: openshift-nmstate
spec:
  channel: stable
  installPlanApproval: Automatic
  name: kubernetes-nmstate-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
```

### Create the Operator Group

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  annotations:
    olm.providedAPIs: NMState.v1.nmstate.io
  generateName: openshift-nmstate-
  name: openshift-nmstate-tn6k8
  namespace: openshift-nmstate
spec:
  targetNamespaces:
  - openshift-nmstate
```

### Create an Empty VLAN on Top of ens1f1 Using Network Bridge

#### Create the Network Bridge

```yaml
Create the network bridge:
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: linux-bridge
spec:
  desiredState:
    interfaces:
    - bridge:
        options:
          stp:
            enabled: false
        port:
        - name: ens1f1
          vlan: {}
      description: Linux bridge on secondary device
      ipv4:
        dhcp: true
        enabled: true
      name: linbridge
      state: up
      type: linux-bridge
  nodeSelector:
    node-role.kubernetes.io/worker: ""
```

#### Create the network attachment definition:
```yaml
apiVersion: "k8s.cni.cncf.io/v1"
kind: NetworkAttachmentDefinition
metadata:
  name: br-vlan
spec:
  config: '{
    "cniVersion": "0.3.1",
    "name": "br-vlan-nad",
    "type": "cnv-bridge",
    "bridge": "linbridge",
    "macspoofchk": true,
    "vlan": 1
  }'
Copy snippet
VM YAML example:

 volumes:
      - name: node-os-vm-root-disk
        dataVolume:
          name: node-os-vm-root-disk
      - name: cloudinitdisk
        cloudInitNoCloud:
          networkData: |
            version: 2
            ethernets:
              enp2s0:
                addresses: [ 172.17.0.1/16 ]
          userData: |
            runcmd:
              - ["sysctl", "-w", "net.ipv6.conf.enp2s0.disable_ipv6=1"]
```
### Migration Settings
We also edited the hyperconverged-cluster-operator:
```yaml
oc edit hco -n openshift-cnv kubevirt-hyperconverged
```
#### And set the following migration settings in order to increase the amount of parallel migrations:

```yaml
  liveMigrationConfig:
    completionTimeoutPerGiB: 800 
    parallelMigrationsPerCluster: 25 # default 5
    parallelOutboundMigrationsPerNode: 5 # default 2
    progressTimeout: 150
```
