---
layout: post
title: CKA 자격 시험 준비를 위한 Kubernetes 정리1 
category: [k8s]
tags: [k8s, cka, kubernetes]
redirect_from:

- /2021/08/30/

---

오늘부터 CKA 자격증 시험을 준비해 보려고 한다. 나의 공부 계획이다.  
1. 쿠버네티스 온라인 강의 수강완료([대세는 쿠버네티스[초급~중급]](https://www.inflearn.com/course/%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%EA%B8%B0%EC%B4%88))
2. Udemy 기출문제 풀이 2회 풀기([Certified Kubernetes Administrator (CKA) with Practice Tests](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/))  

쿠버네티스 시험은 2회까지 볼 수 있기때문에 일단 위의 계획대로 시험을 보고 불합격이 되면 다음 플랜을 세워야 할 것 같다. 여기서는 인프런 강의 실습 내용을 따라해 보면서 정리해 나갈 예정이다.  

# 기본 오브젝트  
## Pod  
파드는 하나 이상의 컨테이너의 그룹이고 쿠버네티스에서 생성하고 관리할 수 있는 배포 가능한 가장 작은 컴퓨팅 단위이다.  

### 1-1. Pod Multi Container Yaml 실행
강의에서는 kubetm/p8000, kubetm/p8080 Nodejs Image를 사용했는데 여기서는 동일한 기능을 하는 coolguy239/p8000, coolguy239/p8080 Springboot Image를 Docker Hub에 업로드해서 사용할 예정이다.  
파드내의 Continer는 Multi로 구성이 가능하지만 동일한 port를 사용할 수 없다.  
```yaml
kubectl apply -f - <<EOF
 apiVersion: v1
 kind: Pod
 metadata:
   name: pod-multi-container
 spec:
   containers:
   - name: container1
     image: coolguy239/p8000
     ports:
     - containerPort: 8000
   - name: container2
     image: coolguy239/p8080
     ports:
     - containerPort: 8080
EOF
```  
### 1-2. Pod Multi Container Yaml 결과확인  
> kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.  
> OCI runtime exec failed: exec failed: container_linux.go:380: starting container process caused: exec: "/bin/bash": stat /bin/bash: no such file or directory: unknown  
> command terminated with exit code 126    
>   
> **Docker Image가 Alpine이라면 /bin/bash를 지원하지 않을 수 있다. 대신 /bin/sh를 사용한다.**    
> Docker build 시 Dockerfile의 jdk 설정이 FROM openjdk:11-jdk로 하면 /bin/bash를 지원  
> Docker build 시 Dockerfile의 jdk 설정이 FROM lpicanco/java11-alpine 로 하면 /bin/sh 지원  

Pod의 Container에 접속해서 호출  
```shell
$ kubectl exec pod-multi-container -c container1 -it /bin/sh  
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
# curl localhost:8000
8000

$ kubectl exec pod-multi-container -c container2 -it /bin/sh  
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
# curl localhost:8080
8080
```  

### 2-1. ReplicationController Yaml 실행
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ReplicationController
metadata:
  name: rpc
spec:
  replicas: 2
  selector:
    label1: labelVal1
    label2: labelVal2
  template:
    metadata:
      labels:
        label1: labelVal1
        label2: labelVal2
    spec:
      containers:
        - name: container
          image: coolguy239/init
EOF
```

### 2-2. ReplicationController Yaml 결과확인  
ReplicationController의 경우 pod를 삭제하면 pod가 재생성 되면서 pod명과 IP가 변경된다.  
```shell
$ kubectl get pod -o wide
NAME                  READY   STATUS    RESTARTS      AGE   IP              NODE          NOMINATED NODE   READINESS GATES
pod-multi-container   2/2     Running   4 (36m ago)   70m   20.100.194.66   k8s-worker1   <none>           <none>
rpc-2k2sx             1/1     Running   0             37s   20.110.126.8    k8s-worker2   <none>           <none>
rpc-6jdsh             1/1     Running   0             37s   20.110.126.7    k8s-worker2   <none>           <none>


$ kubectl delete pod rpc-6jdsh
pod "rpc-6jdsh" deleted

$ kubectl get pod -o wide
NAME                  READY   STATUS    RESTARTS      AGE     IP              NODE          NOMINATED NODE   READINESS GATES
pod-multi-container   2/2     Running   4 (38m ago)   72m     20.100.194.66   k8s-worker1   <none>           <none>
rpc-2k2sx             1/1     Running   0             2m21s   20.110.126.8    k8s-worker2   <none>           <none>
rpc-65vc8             1/1     Running   0             36s     20.110.126.9    k8s-worker2   <none>           <none>
```

### 3-1. Pod의 labels를 이용해 Service 연결 Yaml 실행  
Service에 연결할 Pod 생성  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod01
  labels:
    profile: local
    team: dev1
spec:
  containers:
  - name: container
    image: coolguy239/init
EOF  

$ kubectl get pod -o wide 
NAME                  READY   STATUS    RESTARTS      AGE   IP              NODE          NOMINATED NODE   READINESS GATES
pod-multi-container   2/2     Running   4 (55m ago)   89m   20.100.194.66   k8s-worker1   <none>           <none>
pod01                 1/1     Running   0             99s   20.100.194.68   k8s-worker1   <none>           <none>
rpc-2k2sx             1/1     Running   0             19m   20.110.126.8    k8s-worker2   <none>           <none>
rpc-65vc8             1/1     Running   0             17m   20.110.126.9    k8s-worker2   <none>           <none>
```  

Pod에 연결할 Service 생성  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: pod01-svc
spec:
  selector:
    profile: local
  ports:
  - port: 8080
EOF

kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE     SELECTOR
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    4h15m   <none>
pod01-svc    ClusterIP   10.96.109.213   <none>        8080/TCP   2m11s   profile=local
```

### 3-2. Pod의 labels를 이용해 Service 연결 Yaml 결과확인  
Service의 CLUSTER-IF의 8080 PORT로 API 호출 결과
```shell
$ curl 10.96.109.213:8080/init
This is InitController..
```  

### 4-1. Pod nodeSelector Yaml 실행  
Kubernetes Cluster 내에서 Pod가 특정 Node에 생성되도록 하고 싶다면 spec.nodeSelector 를 이용하면 된다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-node-selector
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1
  containers:
  - name: con-node-selector
    image: coolguy239/init
EOF
```
### 4-2. Pod nodeSelector Yaml 결과확인
spec.nodeSelector에서 설정한 k8s-worker1 노드에 파드 생성확인
```shell
[root@k8s-master ~]# kubectl get pod -o wide
NAME                READY   STATUS    RESTARTS   AGE     IP              NODE          NOMINATED NODE   READINESS GATES
pod-node-selector   1/1     Running   0          3m37s   20.100.194.65   k8s-worker1   <none>           <none>
```  

### 5-1. Pod requests,limits Yaml 실행
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-resource2
spec:
  containers:
  - name: container
    image: kubetm/init
    resources:
      requests:
        memory: 0.5Gi
      limits:
        memory: 0.5Gi
EOF
```
### 5-3. Pod requests,limits Yaml 결과확인
node의 describe 명령으로 node내의 resource 상태를 확인할 수 있다.  
```shell
$ kubectl describe node k8s-worker1

...
Non-terminated Pods:          (3 in total)
  Namespace                   Name                 CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                 ------------  ----------  ---------------  -------------  ---
  default                     pod-resource2        0 (0%)        0 (0%)      2Gi (74%)        3Gi (111%)     16m
  kube-system                 calico-node-cplnm    250m (12%)    0 (0%)      0 (0%)           0 (0%)         15h
  kube-system                 kube-proxy-sfkdh     0 (0%)        0 (0%)      0 (0%)           0 (0%)         15h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests    Limits
  --------           --------    ------
  cpu                250m (12%)  0 (0%)
  memory             2Gi (74%)   3Gi (111%)
  ephemeral-storage  0 (0%)      0 (0%)
  hugepages-2Mi      0 (0%)      0 (0%)
...

```  

쿠버네티스가 Pod를 스케줄링할 때 노드마다 점수를 매겨서 가장 높은 점수가 있는 노드에 파드를 생성한다. 현재는 노드의 리소스 양을 더 적게 사용하고 있는 k8s-worker1 노드에 자원이 할당되었다.

> CPU와 Memory의 request,limits    
> 파드의 CPU가 limits 수치까지 올라갔다고 해서 항상 reuqests 수치까지 낮추는 것이 아니고, 노드의 할당된 CPU, Memory 자원을 넘어섰을 때 동작한다. 노드의 파드들이 노드의 자원을 모두 사용하고 더 많은 자원을 요구하게 되면 CPU의 경우는 limits 수치의 파드들을 requests 수치까지 떨어뜨리게 하고 Memory의 경우는 limits까지 올라간 파드들을 재기동한다.   

## Service  
Service, Pod 모두 Cluster IP를 가지고 있다. 그러다면 Pod에 직접 연결을 하면 되는데 왜 Service를 통해서 Pod을 하는 것일까? Pod는 시스템,성능 장애로 언제든지 죽고 재생성 되도록 설계가 되어있는 Object이다. 그리고 Pod는 재생성되면서 IP가 변경이 되기 때문에 신뢰성이 떨어진다. 그래서 서비스를 통해 Pod에 접근을 한다.  

### ClusterIP
ClusterIP는 쿠버네티스 클러스터 내에서만 접근이 가능하다. 외부에서 접근할 수 없고 인가된 사용자만 접근이 가능하다.  
여기서는 아래와 같은 시나리오로 테스트를 진행할 예정이다.  

Service(ClusterIP),Pod 생성  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
  labels:
     app: pod
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1
  containers:
  - name: container
    image: coolguy239/app
    ports:
    - containerPort: 8080
EOF
```  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: service-01
spec:
  selector:
    app: pod
  ports:
    - port: 9000
      targetPort: 8080
EOF
```  

쿠버네티스 클러스터 내부에서 Service의 IP,PORT로 접속 테스트   
   hostname API는 String값의 hostname을 리턴한다.  
```shell
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    20h
service-01   ClusterIP   10.102.133.92   <none>        9000/TCP   3m49s

$ curl 10.102.133.92:9000/hostname
pod-01
```  

Pod 삭제   
```shell
$ kubectl delete pod pod-01
pod "pod-01" deleted
```
Pod 생성   
```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
  labels:
     app: pod
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1
  containers:
  - name: container
    image: coolguy239/app
    ports:
    - containerPort: 8080
EOF
```  

쿠버네티스 클러스터 내부에서 Service의 IP,PORT로 접속 테스트  
```shell
$ curl 10.102.133.92:9000/hostname
pod-01
```  

### NodePort
NodePort의 경우 Pod가 존재하는 Node에만 설정한 Port가 열리는 게 아니라 쿠버네티스 클러스터 내의 모든 Node의 Port가 오픈된다. NodePort type의 nodePort 속성은 Optional이고 값이 없다면 30000~32767 범위내에서 자동 생성된다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: service-01
spec:
  selector:
    app: pod
  ports:
    - port: 9000
      targetPort: 8080
      nodePort: 30000
  type: NodePort    
  externalTrafficPolicy: Local
EOF
```
#### externalTrafficPolicy 옵션 없거나 externalTrafficPolicy: cluster로 설정한 경우  
k8s-worker1, k8s-worker2 IP의 nodePort로 접근이 가능하다. 클러스터 내부는 아니지만 같은 네트워크인 로컬PC에서 호출한 결과이다.    
```shell
C:\k8s>curl 192.168.56.31:30000/hostname
pod-01
C:\k8s>curl 192.168.56.32:30000/hostname
pod-01
```  

#### externalTrafficPolicy: local로 설정한 경우     
k8s-worker1 노드에 Pod가 생성된 것을 확인했고 k8s-worker2 nodePort로는 접근이 안된다.  
```shell
kubectl get pod -o wide
NAME     READY   STATUS    RESTARTS   AGE     IP              NODE          NOMINATED NODE   READINESS GATES
pod-01   1/1     Running   0          3h36m   20.100.194.69   k8s-worker1   <none>           <none>

C:\k8s>curl 192.168.56.31:30000/hostname
pod-01
C:\k8s>curl 192.168.56.32:30000/hostname
curl: (7) Failed to connect to 192.168.56.32 port 30000: Timed out
```

### Load Balancer  
외부 시스템 노출용도로 사용된다. AWS, GCP, ASURE, Naver 같은 클라우드 환경 내에서는 쿠버네티스 환경내에서는 External IP를 생성해 주는 Plugin이 기본적으로 설치되어 있는데 지금 나와 같은 Window Virtual Box 환경에서는 바로 테스트 하기가 어려워 현재 결과화면만 확인해 보겠다. EXTERNAL-IP가 pending 되어 있는 것을 볼 수 있다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: service-01
spec:
  selector:
    app: pod
  ports:
  - port: 9000
    targetPort: 8080
  type: LoadBalancer
EOF
```
  
```shell
kubectl get svc -o wide
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE     SELECTOR
kubernetes   ClusterIP      10.96.0.1       <none>        443/TCP          25h     <none>
service-01   LoadBalancer   10.102.133.92   <pending>     9000:30000/TCP   5h32m   app=pod
```

## 참고
[KUBETM BLOG](https://kubernetes.io/ko/docs/concepts/workloads/pods/)  
[쿠버네티스 공식사이트](https://kubetm.github.io/k8s/)   
