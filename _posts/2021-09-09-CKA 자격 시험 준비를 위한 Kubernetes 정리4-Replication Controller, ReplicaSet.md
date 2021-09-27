---
layout: post
title: CKA 자격 시험 준비를 위한 Kubernetes 정리4-Replication Controller, ReplicaSet 
category: [k8s]
tags: [k8s, cka, kubernetes]
redirect_from:

- /2021/09/09/

---

오늘은 인프런 강의 대세는 쿠버네티스 강의 중 Replication Controller, ReplicaSet 관련 내용을 실습해 보면서 정리 할 예정이다.  
Replication Controller, ReplicaSet - <https://kubetm.github.io/k8s/04-beginner-controller/replicaset/>   

## Controller 설명  

### Auto Healing  
Node1의 장애로 Pod가 정상적으로 작동하지 못할 때 Controller는 Node2에 Pod를 생성해서 서비스가 정상적으로 유지되도록 한다.    
### Software Update  
여러 Pod에 대한 버전을 업그레이드 해야 할 경우 Controller를 통해 한번에 진행이 가능하고 진행 중 문제가 발생하면 롤백을 할 수도 있다.  
### Auto Scaling
Pod의 Resource가 Limit 상태가 되었을 때 Controller는 상태를 파악해서 Pod를 하나 더 생성해 부하를 분산시켜 준다.  
### Job  
Controller 필요한 순간에만 Pod를 만들어 수행하고 Pod를 삭제해서 효율적인 자원활용을 할 수 있다.  

> Replication Controller는 Deprecated 됨.
> Pod의 label을 설정하고 ReplicationController의 spec.selector에 Pod의 label을 명시해 준다. 
> 그리고 spec.template 하위에 metadata,spec을 명시해 준다.

## Replicaset

### 1. Template, Replicas  

#### 1-1. Pod
terminationGracePeriodSeconds 옵션은 Pod삭제시 Default로는 30초 후에 삭제가 되도록 설정이 되어있다. terminationGracePeriodSeconds: 0을 주면 바로 삭제가 된다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-template-01
  labels:
    type: web
spec:
  containers:
  - name: container
    image: coolguy239/app:v1
  terminationGracePeriodSeconds: 0
EOF
```  

#### 1-2. ReplicaSet
ReplicaSet은 spec.selector에 matchLabels 외에 matchExpression이 제공되어 Pod와의 연결이 용이하다. ReplicaSet을 삭제 시 casecade옵션을 false로 설정하면 ReplicaSet만 삭제되고 기존 생성된 Pod는 남게 된다.   
```shell
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: rset-01
spec:
  replicas: 1
  selector:
    matchLabels:
      type: web
  template:
    metadata:
      name: pod-template-01
      labels:
        type: web
    spec:
      containers:
      - name: container
        image: coolguy239/app:v1
      terminationGracePeriodSeconds: 0
EOF      
```

#### 1-3. Pod 삭제 테스트
Pod를 삭제하면 ReplicaSet Controller에 의해 새로운 파드가 생성되는 것을 확인 할 수 있다. Pod의 이음은 Replicaset이름의 임의의 hash값이 붙어서 생성된다.    
```shell
$ kubectl delete pod rset-01-bvdp2
pod "rset-01-bvdp2" deleted
$ kubectl get pod
rset-01-lp2qx   1/1     Running   0                   6s
```  

#### 1-4. Replicaset Scale UP  
replicas를 3으로 변경하고 Pod를 조회하면 3개의 Pod가 생성된 것을 확인 할 수 있다.  
```shell
$ kubectl scale --replicas=3 rs/rset-01
replicaset.apps/rset-01 scaled
$ kubectl get pod
rset-01-99k26   1/1     Running   0                   21s
rset-01-gltqh   1/1     Running   0                   21s
rset-01-lp2qx   1/1     Running   0                   2m26s
```  

#### 1-5. Pod의 Application 버전업  
edit 명령을 이용해서 원하는 Application의 버전을 coolguy239/app:v1에서 coolguy239/app:v2로 변경 후 pod를 삭제하면 새로운 버전의 Application Pod로 재생성된다.  
```shell
$ kubectl edit rs/rset-01
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"ReplicaSet","metadata":{"annotations":{},"name":"rset-01","namespace":"default"},"spec":{"replicas":1,"selector":{"matchLabels":{"type":"web"}},"template":{"metadata":{"labels":{"type":"web"},"name":"pod-template-01"},"spec":{"containers":[{"image":"coolguy239/app:v1","name":"container"}],"terminationGracePeriodSeconds":0}}}}
  creationTimestamp: "2021-09-04T22:29:59Z"
  generation: 4
  name: rset-01
  namespace: default
  resourceVersion: "88129"
  uid: b5130d2d-f685-4d76-b7b2-495a9db2c180
spec:
  replicas: 3
  selector:
    matchLabels:
      type: web
  template:
    metadata:
      creationTimestamp: null
      labels:
        type: web
      name: pod-template-01
    spec:
      containers:
      - image: coolguy239/app:v2
        imagePullPolicy: IfNotPresent
        name: container
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 0
replicaset.apps/rset-01 edited

$ kubectl delete pod rset-01-99k26 rset-01-gltqh rset-01-lp2qx
pod "rset-01-99k26" deleted
pod "rset-01-gltqh" deleted
pod "rset-01-lp2qx" deleted

$ kubectl get pod | grep rset-01
rset-01-9bhvj   1/1     Running   0                   64s
rset-01-pkrjg   1/1     Running   0                   64s
rset-01-wbb9x   1/1     Running   0                   64s

$ kubectl describe pod rset-01-9bhvj
...
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  96s        default-scheduler  Successfully assigned default/rset-01-9bhvj to k8s-worker2
  Normal  Pulling    <invalid>  kubelet            Pulling image "coolguy239/app:v2"
  Normal  Pulled     <invalid>  kubelet            Successfully pulled image "coolguy239/app:v2" in 5.387833393s
  Normal  Created    <invalid>  kubelet            Created container container
  Normal  Started    <invalid>  kubelet            Started container container
```

### 2. Updating Controller

#### 2-1. ReplicationController 생성
spec.selector에 바로 key: value 형태의 Label로 Pod와 연결
````shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ReplicationController
metadata:
  name: replcon-01
spec:
  replicas: 2
  selector:
    cascade: "false"
  template:
    metadata:
      labels:
        cascade: "false"
    spec:
      containers:
      - name: container
        image: coolguy239/app:v1
EOF

replicationcontroller/replcon-01 created
````  
#### 2-2. ReplicationController 삭제
replicationcontroller를 삭제 할때 casecade 옵션의 value 값으로 orphan을 설정을 주면 replicationcontroller만 삭제되고 pod는 그대로 존재한다.  
```shell
$ kubectl delete replicationcontrollers replcon-01 --cascade=orphan
replicationcontroller "replcon-01" deleted

$ kubectl get replicationcontrollers
No resources found in default namespace.

$ kubectl get pod | grep replcon-01
replcon-01-9629h   1/1     Running   0                   37s
replcon-01-vp7rp   1/1     Running   0                   37s
```  

#### 2-3. ReplicaSet 생성
spec.selector.matchLabels에 key: value 형태로 Pod와 연결
```shell
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replset-01
spec:
  replicas: 2
  selector:
    matchLabels:
      cascade: "false"
  template:
    metadata:
      labels:
        cascade: "false"
    spec:
      containers:
      - name: container
        image: coolguy239/app:v1
EOF

replicaset.apps/replset-01 created
```  


#### 2-4. ReplicaSet 삭제
```shell
$ kubectl delete replicaset replset-01 --cascade=orphan
replicaset.apps "replset-01" deleted

$ kubectl get pod | grep replset-01
replset-01-fbmvw   1/1     Running   0                   17s
replset-01-n8cts   1/1     Running   0                   17s
```

### 3. Selector
ReplicaSet에서는 matchExpressions 보다는 matchLabels를 주로 사용한다. ReplicaSet의 spec.selector.matchLabels은 template 하위에 spec.template.metadata.labels 하위 목록에 포함이 되어야 한다.

#### 3-1. Selector matchLabels 생성오류 예제  
spec.selector.matchLabels의 내용이 spec.template.metadata.labels 하위의 내용에 포함되지 않는 경우
```shell
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replset-02
spec:
  replicas: 1
  selector:
    matchLabels:
      type: web
      ver: v2
  template:
    metadata:
      labels:
        type: web
        ver: v1
        location: dev
    spec:
      containers:
      - name: container
        image: coolguy239/app:v1
      terminationGracePeriodSeconds: 0
EOF

The ReplicaSet "replset-02" is invalid: spec.template.metadata.labels: Invalid value: map[string]string{"location":"dev", "type":"web", "ver":"v1"}: `selector` does not match template `labels`
```  

#### 3-2. Selector matchLabels 생성
```shell
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replset-02
spec:
  replicas: 1
  selector:
    matchLabels:
      type: web
      location: dev
  template:
    metadata:
      labels:
        type: web
        ver: v1
        location: dev
    spec:
      containers:
      - name: container
        image: coolguy239/app:v1
      terminationGracePeriodSeconds: 0
EOF

replicaset.apps/replset-02 created
```  

#### 3-3. Selector matchExpressions 생성
matchExpressions의 내용이 template 하위에 spec.template.metadata.labels 하위 목록에 포함이 되어야 한다.
```shell
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replset-03
spec:
  replicas: 1
  selector:
    matchExpressions:
    - {key: type, operator: In, values: [web]}
    - {key: location, operator: Exists}
  template:
    metadata:
      labels:
        type: web
        ver: v1
        location: dev
    spec:
      containers:
      - name: container
        image: coolguy239/app:v1
      terminationGracePeriodSeconds: 0
EOF
```  


## 참고  
[KUBETM BLOG](https://kubetm.github.io/k8s/04-beginner-controller/replicaset/)     
[쿠버네티스 공식사이트-ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)  
[쿠버네티스 공식사이트-ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)  
