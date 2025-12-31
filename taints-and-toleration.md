## Enabling vLLM workloads land on the right GPU node

To enable vLLM workloads land on the right GPU node, following three things have to be done. 
- Adding taints to `MachineSet`
- Include toleration to the `ClusterPolicy` CRD of NVIDIA GPU operator
- Add the toleration and taints to the hardware profiles of RHOAI and use it while creating model serving deployments

### Adding taints to the MachineSet
```
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  annotations:
    capacity.cluster-autoscaler.kubernetes.io/labels: kubernetes.io/arch=amd64
    machine.openshift.io/GPU: '1'
    machine.openshift.io/memoryMb: '65536'
    machine.openshift.io/vCPU: '16'
  resourceVersion: '132808'
  name: sreips-test-worker
  namespace: openshift-machine-api
  labels:
    machine.openshift.io/cluster-api-cluster: cluster-w9z67-62n7r
spec:
  replicas: 1
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: cluster-w9z67-62n7r
      machine.openshift.io/cluster-api-machineset: sreips-test-worker
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: cluster-w9z67-62n7r
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machine-type: worker
        machine.openshift.io/cluster-api-machineset: sreips-test-worker
    spec:
      lifecycleHooks: {}
      metadata:
        labels:
          node-role.kubernetes.io/worker: ''
          sreips-demo: node-test
      providerSpec:
        value:
          userDataSecret:
            name: worker-user-data
          placement:
            availabilityZone: us-east-2a
            region: us-east-2
          credentialsSecret:
            name: aws-cloud-credentials
          capacityReservationId: ''
          instanceType: g6.4xlarge
          metadata:
            creationTimestamp: null
          blockDevices:
            - ebs:
                encrypted: true
                iops: 0
                kmsKey:
                  arn: ''
                volumeSize: 100
                volumeType: gp2
          securityGroups:
            - filters:
                - name: 'tag:Name'
                  values:
                    - cluster-w9z67-62n7r-node
            - filters:
                - name: 'tag:Name'
                  values:
                    - cluster-w9z67-62n7r-lb
          kind: AWSMachineProviderConfig
          metadataServiceOptions: {}
          tags:
            - name: kubernetes.io/cluster/cluster-w9z67-62n7r
              value: owned
            - name: Stack
              value: ocp4-cluster-wvkq8
            - name: env_type
              value: ocp4-cluster
            - name: guid
              value: df2jq
            - name: owner
              value: unknown
            - name: platform
              value: rhpds
            - name: uuid
              value: 06c9de12-6cc8-5e35-9332-21ba2e616379
          deviceIndex: 0
          ami:
            id: ami-0bc8dda494f111572
          subnet:
            id: subnet-05a9a72c36b6099c5
          apiVersion: machine.openshift.io/v1beta1
          iamInstanceProfile:
            id: cluster-w9z67-62n7r-worker-profile
      taints:
        - effect: NoSchedule
          key: sreips-demo
          value: node-test
```

### Adding toleration to the DaemonSet in ClusterPolicy of GPU operator 
```
apiVersion: nvidia.com/v1
kind: ClusterPolicy
metadata:
  name: gpu-cluster-policy
spec:
  vgpuDeviceManager:
    config:
      default: default
    enabled: true
  migManager:
    config:
      default: all-disabled
      name: default-mig-parted-config
    enabled: true
  operator:
    defaultRuntime: crio
    initContainer: {}
    runtimeClass: nvidia
    use_ocp_driver_toolkit: true
  dcgm:
    enabled: true
  gfd:
    enabled: true
  dcgmExporter:
    config:
      name: ''
    enabled: true
    serviceMonitor:
      enabled: true
  cdi:
    default: false
    enabled: true
  driver:
    licensingConfig:
      nlsEnabled: true
      secretName: ''
    enabled: true
    kernelModuleType: auto
    certConfig:
      name: ''
    useNvidiaDriverCRD: false
    kernelModuleConfig:
      name: ''
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
    virtualTopology:
      config: ''
  devicePlugin:
    config:
      default: ''
      name: ''
    enabled: true
    mps:
      root: /run/nvidia/mps
  gdrcopy:
    enabled: false
  kataManager:
    config:
      artifactsDir: /opt/nvidia-gpu-operator/artifacts/runtimeclasses
  mig:
    strategy: single
  sandboxDevicePlugin:
    enabled: true
  validator:
    plugin:
      env: []
  nodeStatusExporter:
    enabled: true
  daemonsets:
    rollingUpdate:
      maxUnavailable: '1'
    tolerations:
      - effect: NoSchedule
        key: sreips-demo
        operator: Equal
        value: node-test
    updateStrategy: RollingUpdate
  sandboxWorkloads:
    defaultWorkload: container
    enabled: false
  gds:
    enabled: false
  vgpuManager:
    enabled: false
  vfioManager:
    enabled: true
  toolkit:
    enabled: true
    installDir: /usr/local/nvidia
```

### Hardware profiles
```
apiVersion: infrastructure.opendatahub.io/v1
kind: HardwareProfile
metadata:
  annotations:
    opendatahub.io/dashboard-feature-visibility: '[]'
    opendatahub.io/disabled: 'false'
    opendatahub.io/display-name: nvidia-gpu
  name: nvidia-gpu
  namespace: redhat-ods-applications
spec:
  identifiers:
    - defaultCount: 2
      displayName: CPU
      identifier: cpu
      maxCount: 4
      minCount: 1
      resourceType: CPU
    - defaultCount: 16Gi
      displayName: Memory
      identifier: memory
      maxCount: 16Gi
      minCount: 8Gi
      resourceType: Memory
    - defaultCount: 1
      displayName: gpu
      identifier: nvidia.com/gpu
      maxCount: 2
      minCount: 1
      resourceType: Accelerator
  scheduling:
    node:
      nodeSelector:
        node.kubernetes.io/instance-type: g6.4xlarge
      tolerations:
        - effect: NoSchedule
          key: sreips-demo
          operator: Equal
          value: node-test
    type: Node
```
