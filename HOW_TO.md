# Ansible 기반의 Operator Framework 사용 가이드 - Memcached

## 1. Operator SDK 설치

> [operator-sdk 설치 가이드](https://sdk.operatorframework.io/docs/installation/)

환경 변수 설정

```bash
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')
```

operator-sdk 설치

```bash
export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.39.2
curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
sudo install operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk
```

> 참고  
> 1.31 버전 까지는 Apple Silicon용 Ansible Operator가 빌드되었는데, 1.32 부터 빌드하지 않음
> - https://github.com/operator-framework/operator-sdk/releases  
> - https://github.com/operator-framework/operator-sdk/releases/tag/v1.31.0  
> - https://github.com/operator-framework/operator-sdk/releases/tag/v1.32.0  

## 2. Ansible Operator 프로젝트 생성 및 구성

프로젝트 디렉토리 생성

```bash
mkdir memcached-operator
cd memcached-operator
```

프로젝트 초기화

```bash
operator-sdk init --plugins=ansible --domain example.com
```

API 생성

```bash
operator-sdk create api --group cache --version v1alpha1 --kind Memcached --generate-role
```

> [프로젝트 레이아웃](https://sdk.operatorframework.io/docs/overview/project-layout/)

`roles/memcached/tasks/main.yml` 파일 수정

```yaml
---
- name: start memcached
  kubernetes.core.k8s:
    state: present
    definition:
      kind: Deployment
      apiVersion: apps/v1
      metadata:
        name: '{{ ansible_operator_meta.name }}'
        namespace: '{{ ansible_operator_meta.namespace }}'
        labels:
          app.kubernetes.io/name: memcached-operator
          app.kubernetes.io/instance: '{{ ansible_operator_meta.name }}'
      spec:
        replicas: "{{size}}"
        selector:
          matchLabels:
            app.kubernetes.io/name: memcached-operator
            app.kubernetes.io/instance: '{{ ansible_operator_meta.name }}'
        template:
          metadata:
            labels:
              app.kubernetes.io/name: memcached-operator
              app.kubernetes.io/instance: '{{ ansible_operator_meta.name }}'
          spec:
            containers:
            - name: memcached
              command:
              - memcached
              - -o
              - modern
              - -v
              image: "docker.io/memcached:1.6-alpine"
              ports:
                - name: memcached
                  protocol: TCP
                  containerPort: 11211

- name: start memcached
  kubernetes.core.k8s:
    state: present
    definition:
      kind: Service
      apiVersion: v1
      metadata:
        name: "{{ ansible_operator_meta.name }}"
        namespace: "{{ ansible_operator_meta.namespace }}"
        labels:
          app.kubernetes.io/name: memcached-operator
          app.kubernetes.io/instance: '{{ ansible_operator_meta.name }}'
      spec:
        type: "{{ service_type }}"
        selector:
          app.kubernetes.io/name: memcached-operator
          app.kubernetes.io/instance: '{{ ansible_operator_meta.name }}'
        ports:
          - name: memcached
            protocol: TCP
            port: "{{service_port}}"
            targetPort: 11211

- name: Get memcached pods infos
  kubernetes.core.k8s_info:
    api_version: v1
    kind: Pod
    namespace: '{{ ansible_operator_meta.namespace }}'
    label_selectors:
      - app.kubernetes.io/name=memcached-operator
  register: pods_info

- name: Set node names list
  set_fact:
    node_names: "{{ pods_info.resources | selectattr('status.phase', 'equalto', 'Running') | map(attribute='spec.nodeName') | list }}"
- debug:
    msg: "node_names: {{ node_names }}"

- name: Set current size from pod count
  set_fact:
    current_size: "{{ pods_info.resources | selectattr('status.phase', 'equalto', 'Running') | list | length }}"
- debug:
    msg: "current_size: {{ current_size }}"

- name: Set memcached state
  set_fact:
    memcached_state: >-
      {%- if current_size | int == size | int -%}
      Green
      {%- elif current_size | int > 0 -%}
      Yellow
      {%- else -%}
      Red
      {%- endif -%}
- debug:
    msg: "memcached_state: {{ memcached_state }}"

- name: Update memcached status
  operator_sdk.util.k8s_status:
    api_version: cache.example.com/v1alpha1
    kind: Memcached
    name: "{{ ansible_operator_meta.name }}"
    namespace: "{{ ansible_operator_meta.namespace }}"
    force: true
    status:
      state: "{{ memcached_state | trim }}"
      nodes: "{{ node_names }}"
      current_size: "{{ current_size }}"
```

`roles/memcached/defaults/main.yml` 파일 수정

```yaml
---
# defaults file for Memcached
size: 1
service_port: 11211
service_type: ClusterIP
```

리소스 배포 예시 `config/samples/cache_v1alpha1_memcached.yaml` 파일 수정

```yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  labels:
    app.kubernetes.io/name: memcached-operator
    app.kubernetes.io/managed-by: kustomize
  name: memcached-example
spec:
  size: 3
  service_port: 11211
  service_type: ClusterIP
```

CRD 정의 `config/crd/bases/cache.example.com_memcacheds.yaml` 파일 수정

```yaml
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: memcacheds.cache.example.com
spec:
  group: cache.example.com
  names:
    kind: Memcached
    listKind: MemcachedList
    plural: memcacheds
    singular: memcached
    shortNames:
    - mcd
  scope: Namespaced
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        description: Memcached is the Schema for the memcacheds API
        properties:
          apiVersion:
            description: 'APIVersion defines the versioned schema of this representation
              of an object. Servers should convert recognized schemas to the latest
              internal value, and may reject unrecognized values. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources'
            type: string
          kind:
            description: 'Kind is a string value representing the REST resource this
              object represents. Servers may infer this from the endpoint the client
              submits requests to. Cannot be updated. In CamelCase. More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds'
            type: string
          metadata:
            type: object
          spec:
            description: Spec defines the desired state of Memcached
            type: object
            properties:
              size:
                description: Size is the number of Memcached instances
                type: integer
                minimum: 1
              port:
                description: Port is the port number of Memcached instances
                type: integer
                default: 11211
              serviceType:
                description: ServiceType is the type of service to create for Memcached
                type: string
                default: ClusterIP
            required:
              - size
          status:
            description: Status defines the observed state of Memcached
            type: object
            properties:
              state:
                description: Status of the Memcached instances (Green, Yellow, Red)
                type: string
                enum: ["Green", "Yellow", "Red"]
              nodes:
                description: List of nodes where Memcached instances are running
                type: array
                items:
                  type: string
              current_size:
                description: Current size of the Memcached instances
                type: string
        type: object
    served: true
    storage: true
    subresources:
      status: {}
    additionalPrinterColumns:
    - name: STATE
      type: string
      description: The state of the Memcached instances (Green, Yellow, Red)
      jsonPath: .status.state
    - name: AGE
      type: date
      description: The age of the resource
      jsonPath: .metadata.creationTimestamp
```

`watches.yaml` 파일 수정

```yaml
---
# Use the 'create api' subcommand to add watches to this file.
- version: v1alpha1
  group: cache.example.com
  kind: Memcached
  role: memcached
  reconcilePeriod: 60s
  manageStatus: False
  watchDependentResources: True
# +kubebuilder:scaffold:watch
```

## 3. Ansible Operator 이미지 빌드

`Makefile` 파일 수정

```bash
...
VERSION ?= X.X.X
...
IMAGE_TAG_BASE ?= <REGISTRY>/<ACCOUNT>/memcached-operator
...
IMG ?= ${IMAGE_TAG_BASE}:${VERSION}
...
PLATFORMS ?= linux/arm64,linux/amd64
...
```

컨테이너 저장소 인증

```bash
docker login 
```

이미지 빌드 및 푸시

```bash
make docker-build docker-push
```

또는 크로스 플랫폼 이미지 빌드 및 푸시

```bash
make docker-buildx
```

## 4. Operator 배포

로컬에 Operator 배포

```bash
make deploy
```

추후 배포시 사용할 Operator 메니페스트 파일 생성

```bash
bin/kustomize build config/default > memcached-operator.yaml
```

Operator 배포 확인

```bash
kubectl get crds
kubectl api-resources --api-group=cache.example.com
kubectl get deployment -n memcached-operator-system
kubectl get pods -n memcached-operator-system -w
```

Memcached 리소스 배포

```bash
kubectl apply -f config/samples/cache_v1alpha1_memcached.yaml
```

Memcached 리소스 확인

```bash
kubectl get memcacheds.cache.example.com
kubectl get mcd
kubectl get deployments.apps
kubectl get pods 
kubectl get svc,endpoints
```

Memcached 리소스 크기 변경

```bash
kubectl patch memcached memcached-sample -p '{"spec":{"size": 5}}' --type=merge
kubectl get mcd
kubectl get deploy
kubectl get pods
```

memcached.cache.example.com 리소스 전체 구조

```bash
kubectl get memcacheds.cache.example.com memcached-example -o yaml
```

```yaml
apiVersion: cache.example.com/v1alpha1
kind: Memcached
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"cache.example.com/v1alpha1","kind":"Memcached","metadata":{"annotations":{},"labels":{"app.kubernetes.io/managed-by":"kustomize","app.kubernetes.io/name":"memcached-operator"},"name":"memcached-example","namespace":"default"},"spec":{"servicePort":11211,"serviceType":"ClusterIP","size":3}}
  creationTimestamp: "2025-04-04T07:50:54Z"
  generation: 2
  labels:
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/name: memcached-operator
  name: memcached-example
  namespace: default
  resourceVersion: "141605"
  uid: b9972c03-720f-48e5-8bff-e98df9e58380
spec:
  servicePort: 11211
  serviceType: ClusterIP
  size: 5
status:
  current_size: "5"
  nodes:
  - kube-node1
  - kube-node3
  - kube-node2
  - kube-node3
  - kube-node1
  state: Green
```
