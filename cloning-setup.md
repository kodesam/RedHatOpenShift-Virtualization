
```markdown
# Cloning Setup

## Templates

Note that all the OS templates we used are the default templates that are available through the Red Hat OpenShift Virtualization templates wizard, with a few custom changes to the network.

### Red Hat Linux

The template being used can be retrieved by:

```bash
oc get templates -n openshift rhel9-server-medium -o yaml
```

### Pod

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: pod-name
  namespace: pods-namespace
  labels:
    name: vdpod-density
spec:
  nodeSelector:
    node-role.kubernetes.io/worker: ""
  restartPolicy: "Always"
  containers:
  - name: pod-name
    image: gcr.io/google_containers/pause-amd64:3.0
    imagePullPolicy: IfNotPresent
    securityContext:
      privileged: false
```

### Virtual Machine

Below is a simple VM YAML as a reference that utilized both the Multus secondary network and the golden snapshot we just created for the VM:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: rhel001
spec:
  running: false
  template:
    metadata:
      labels:
        kubevirt.io/vm: rhel001
    spec:
      terminationGracePeriodSeconds: 0
      evictionStrategy: LiveMigrate
      domain:
        cpu:
          cores: 1
          sockets: 1
          threads: 1
        devices:
          blockMultiQueue: false
          disks:
          - disk:
              bus: virtio
            name: rhel001
            dedicatedIOThread: false
          interfaces:
          - bridge: {}
            model: virtio
            name: nic-0
          networkInterfaceMultiqueue: true
          rng: {}
        machine:
          type: pc-q35-rhel9.2.0
        resources:
          requests:
            cpu: "1"
            memory: 2G
      networks:
      - multus:
          networkName: br-vlan
        name: nic-0
      volumes:
      - dataVolume:
          name: rhel001
        name: rhel001
  dataVolumeTemplates:
  - metadata:
      name: rhel001
    spec:
      pvc:
        accessModes:
        - ReadWriteMany
        resources:
          requests:
            storage: 21Gi
        volumeMode: Block
        storageClassName: ocs-external-storagecluster-ceph-rbd
```

## VMs Deployment

With the snapshot golden image and VM template prepared, we are now able to initiate the cloning process for the desired number of VMs. We successfully accomplished the deployment of 6,000 VMs within 40 minutes, and the complete cluster configuration within 4 hours. The Results section provides additional details.

Note that while deploying VMs, it's important to exercise caution and be mindful not to surpass the capacity limit of the RHCS storage pool. Additionally, consider the possibility that certain VMs might necessitate more storage in subsequent instances, potentially leading to storage reaching its full capacity. This precaution is essential to ensure a smooth and well-managed deployment process.

## Creating Snap Source

We start by importing QCOW image from any accessible remote as a DV (Data Volume). Once the import was completed, we can now proceed to create a snapshot golden image which will be used as a source for all other clones:

```yaml
apiVersion: cdi.kubevirt.io/v1alpha1
kind: DataVolume
metadata:
  name: golden-rhel9.2
spec:
  source:
    http:
      url: http://internel.server.com/ISO/golden_rhel9_2.qcow2
  pvc:
    accessModes:
      - ReadWriteMany
    resources:
      requests:
        storage: 21Gi
    volumeMode: Block
    storageClassName: ocs-external-storagecluster-ceph-rbd
```

Choose the required snap class by running:

```bash
oc get volumesnapshotclass
```

Below are the available volume snapshot classes for the cluster:

- `ocs-external-storagecluster-cephfsplugin-snapclass`
- `ocs-external-storagecluster-rbdplugin-snapclass`

The next step is to create a snapshot of the golden image using the DV we imported:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: golden-rhel
  namespace: default
spec:
  volumeSnapshotClassName: ocs-external-storagecluster-rbdplugin-snapclass
  source:
    persistentVolumeClaimName: golden-rhel9.2
```

Now we can create `clone_vm.yaml` and use it to create as many snapshots of the golden image as we require:

```yaml
apiVersion: template.openshift.io/v1
kind: Template
objects:
- apiVersion: cdi.kubevirt.io/v1beta1
  kind: DataVolume
  metadata:
    name: ${VM}
    namespace: default
  spec:
    source:
      snapshot:
        namespace: default
        name: golden-rhel
    storage:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 21Gi
      storageClassName: ocs-external-storagecluster-ceph-rbd
      volumeMode: Block
parameters:
- description: root-disk-name
  name: VM
  value: "rhel"
```

By running the following command, we create a VM clone named `rhel0001`. We do that by replacing the value of the `VM` parameters with the new value mentioned below - `rhel0001`. This process can be repeated as required:

```bash
oc process -f templates/clone_vm.yaml -p VM=rhel0001 | oc create -f -
```

Once the VMs are sorted the way we need them to be, it’s time to test them and ensure they are performing as expected.
```

Feel free to adjust the content as needed! If you have any questions or need further assistance, just let me know.
