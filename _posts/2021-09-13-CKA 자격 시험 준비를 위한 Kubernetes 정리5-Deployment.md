---
layout: post
title: CKA 자격 시험 준비를 위한 Kubernetes 정리5-Deployment  
category: [k8s]
tags: [k8s, cka, kubernetes]
redirect_from:

- /2021/09/13/

---

오늘은 인프런 강의 대세는 쿠버네티스 강의 중 Deployment 관련 내용을 실습해 보면서 정리 할 예정이다.  
Deployment - <https://kubetm.github.io/k8s/04-beginner-controller/deployment/>   
## Controller - Deployment 설명
아래 설명들은 Deployment로 Pod 2개를 운영하고 있다고 가정한다.  

### Recreate  
v1 Pod를 v2로 업그레이드시 v1 Pod Downtime이 생기고 v2 Pod가 생성된다. 일시적인 정지가 가능한 서비스일 때 사용한다.  
Pod가 2대(v1)->0대->2대(v2)로 변경된다.  

### Rolling Update    
일시적으로 Pod가 3대까지 증가한다. 배포 중간 추가적인 자원을 필요로 한다. Downtime도 없다.  
Pod가 2대(v1)->3대(v1:2대, v2:1대)->2대(v1:1대,v1:1대)->2대(v1:1대,v1:2대)->2대(v1:2대)

### Blue/Green  
Service의 Label 설정을 통한 배포방법이다. 운영 중인 v1 Pod 2대 외에 v2 Pod 2대를 더 띄운다. 일시적으로 4대의 Pod의 자원이 필요하게 되고 Service에 설정된 v1 Pod를 바라보게 설정한 Label을 v2 Pod를 바라보는 Label로 변경한다. 많이 사용되는 배포 방법이고 Downtime이 없고 Rollback이 용이하다.
Pod가 2대(v1)->4대(v1:2대,v2:2대)->서비스 정상확인->2대(v2:2대)  

### Canary  
카나리아라는 새의 이름에서 유래된다. 카나리아는 유독가스에 굉장히 민감한 동물로 석탄 광산에서 유독가스 누출의 위험을 알리는 용도로 사용되었다.  
Service에 v1 Pod 2대에 v2 Pod 1대를 추가로 설정해서 불특정 유저의 유입으로 v2 Pod가 문제가 없는지 확인하고 확인이 되었다면 v1 Pod를 v2로 전환한다.  
Pod가 2대(v1)->3대(v1:2대,v2:1대)->2대(v2:2대)

## Deployment  
Deployment는 selector,replicas,template 값을 갖고 있고 직접 Pod를 만들지 않고 ReplicaSet을 만들기 위한 값이다. ReplicaSet이 Pod를 만들게 된다.  
### 1. Recreate  
spec.strategy.type: Recreate로 설정, spec.strategy.revisionHistoryLimit: 1은 ReplicaSet의 replicas의 숫자값이 0인 History를 1개만 남기겠다는 의미이고 Optional 값으로 default 값은 10이다.

#### 1-1. Deployment(Recreate)  
Deployment, ReplicaSet으로 Pod를 생성하게 되면 내부적으로 pod-template-hash라는 label이 생성되어 기존에 사용중이던 Pod와의 구분값으로 사용된다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment-01
spec:
  selector:
    matchLabels:
      type: app
  replicas: 2
  strategy:
    type: Recreate
  revisionHistoryLimit: 1
  template:
    metadata:
      labels:
        type: app
    spec:
      containers:
      - name: container
        image: coolguy239/app:v4
      terminationGracePeriodSeconds: 10
EOF
deployment.apps/deploy-01 created

$ kubectl get pod --show-labels
NAME                         READY   STATUS    RESTARTS      AGE     LABELS
deploy-01-67b6f8695b-bx5jw   1/1     Running   0             4m18s   pod-template-hash=67b6f8695b,type=app
deploy-01-67b6f8695b-h5x2r   1/1     Running   0             4m18s   pod-template-hash=67b6f8695b,type=app
```  

#### 1-2. Service(Recreate)  
라벨이 type=app인 pod를 조회해서 Service와 연결이 되었는지 확인한다. Endpoints를 보면 두개의 Pod IP가 연결된 것을 확인할 수 있다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: svc-01
spec:
  selector:
    type: app
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 8080
EOF
service/svc-01 created

$ kubectl get pod -l type=app -o wide
NAME                         READY   STATUS    RESTARTS   AGE     IP               NODE          NOMINATED NODE   READINESS GATES
deploy-01-67b6f8695b-bx5jw   1/1     Running   0          7m57s   20.100.194.73   k8s-worker1   <none>           <none>
deploy-01-67b6f8695b-h5x2r   1/1     Running   0          7m57s   20.100.194.76   k8s-worker1   <none>           <none>

$ kubectl describe svc svc-01
Name:              svc-01
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          type=app
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.108.136.144
IPs:               10.108.136.144
Port:              <unset>  8080/TCP
TargetPort:        8080/TCP
Endpoints:         20.100.194.73:8080,20.100.194.76:8080
Session Affinity:  None
Events:            <none>
```  

#### 1-3. Service의 Cluster IP를 반복호출
Service를 반복호출 하고 Deployment의 version 정보를 Edit한다.(coolguy239/app:v3 -> coolguy239/app:v4)
```shell
$ while true; do curl 10.108.136.144:8080/version; sleep 1; done
v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3...

$ kubectl edit deployment deployment-01
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"name":"deployment-01","namespace":"default"},"spec":{"replicas":2,"revisionHistoryLimit":1,"selector":{"matchLabels":{"type":"app"}},"strategy":{"type":"Recreate"},"template":{"metadata":{"labels":{"type":"app"}},"spec":{"containers":[{"image":"coolguy239/app:v3","name":"container"}],"terminationGracePeriodSeconds":10}}}}
  creationTimestamp: "2021-09-13T15:16:47Z"
  generation: 1
  name: deployment-01
  namespace: default
  resourceVersion: "109771"
  uid: ebcb978b-cd79-41e7-8e95-147c7e921f9d
spec:
  progressDeadlineSeconds: 600
  replicas: 2
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      type: app
  strategy:
    type: Recreate
  template:
    metadata:
      creationTimestamp: null
      labels:
        type: app
    spec:
      containers:
      - image: coolguy239/app:v4
        imagePullPolicy: IfNotPresent
        name: container
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        
deployment.apps/deployment-01 edited
```  

#### 1-4. Deployment 결과확인
strategy가 Recreate로 설정을 했기 때문에 중간에 Downtime이 발생하는 것을 확인할 수 있다.  
```shell
v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3v3curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
curl: (7) Failed connect to 10.108.136.144:8080; Connection refused
v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4v4
```  

#### 1-5. 배포 History 확인 및 Undo
버전을 v4에서 v5 올려서 배포하고 다시 history 조회하면 revisionHistoryLimit: 1 설정을 했기 때문에 현재 revision 이외에 하나만 남는 것을 확인 할 수 있다.
```shell
$ kubectl rollout history deployment deployment-01
deployment.apps/deployment-01 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>

# kubectl edit를 통해 v4->v5 변경 후 History 조회
$ kubectl rollout history deployment deployment-01
deployment.apps/deployment-01 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
 
$ kubectl rollout undo deployment deployment-01 --to-revision=2
deployment.apps/deployment-01 rolled back

$ kubectl get pod
NAME                             READY   STATUS        RESTARTS   AGE
deployment-01-7c894b6dd4-782db   1/1     Terminating   0          6m16s
deployment-01-7c894b6dd4-wnz8c   1/1     Terminating   0          6m16s

$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
deployment-01-5788495b4c-mvqgh   1/1     Running   0          21s
deployment-01-5788495b4c-np2xw   1/1     Running   0          20s
```

### 2. RollingUpdate  
spec.strategy.type: RollingUpdate - Downtime 없이 전환  
spec.strategy.minReadySeconds: 10 - v1, v2에 Pod가 추가되고 삭제되는 시간(sec)이다.    
```shell

```


### 3. Blue/Green  

## 참고  
[KUBETM BLOG](https://kubetm.github.io/k8s/04-beginner-controller/deployment/)     
[쿠버네티스 공식사이트-Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)  
