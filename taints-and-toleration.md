To enable vLLM workloads land on the right GPU node, following three things have to be done. 
- Adding taints to the MachineSet
- Include toleration to the gpu-cluster-config CRD of NVIDIA GPU operator
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
