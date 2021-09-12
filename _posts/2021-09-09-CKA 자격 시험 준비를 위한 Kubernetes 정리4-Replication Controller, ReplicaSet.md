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
ReplicaSet은 spec.selector에 matchLabels 외에 matchExpression이 제공되어 Pod와의 연결이 용이하다.
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

### 2. Updating Controller
### 3. Selector


## 참고  
[KUBETM BLOG](https://kubetm.github.io/k8s/04-beginner-controller/replicaset/)     
[쿠버네티스 공식사이트-ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)  
[쿠버네티스 공식사이트-ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)  
