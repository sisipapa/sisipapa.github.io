---
layout: post
title: CKA 자격 시험 준비를 위한 Kubernetes 정리3-Namespace,ResourceQuota,LimitRange 
category: [k8s]
tags: [k8s, cka, kubernetes]
redirect_from:

- /2021/09/08/

---

오늘은 인프런 강의 대세는 쿠버네티스 강의 중 Namespace,ResourceQuota,LimitRange 관련 내용을 실습해 보면서 정리 할 예정이다.  
Namespace,ResourceQuota,LimitRange - <https://kubetm.github.io/k8s/03-beginner-basic-resource/namespace/>   

## Namespace
하나의 Namespace 안에서 같은 Type의 오브젝트 이름을 사용할 수 없다. Node, PV와 같은 공용 오브젝트를 제외하고 pod,service 등의 대부분의 오브젝트는 동일 Namespace 에서만 사용이 가능한다.   
### 1. Namespace, Pod, Service 연결
#### 1-1. Namespace
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nm-01
EOF
```  
#### 1-2. Pod
Namespace nm-01로 지정해 주고, Service 연결을 위해 labels(app: pod)를 추가
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
  namespace: nm-01
  labels:
    app: pod
spec:
  containers:
  - name: container
    image: coolguy239/app
    ports:
    - containerPort: 8080
EOF
```  
#### 1-3. Service
Namespace nm-01로 지정, Pod(pod-01)를 연결하기 위해 spec.selector(app: pod) 설정  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: svc-01
  namespace: nm-01
spec:
  selector:
    app: pod
  ports:
  - port: 9000
    targetPort: 8080
EOF
```  

#### 1-4. pod로 접속확인
```shell
$ kubectl get pod -n nm-01 -o wide
NAME     READY   STATUS    RESTARTS   AGE   IP              NODE          NOMINATED NODE   READINESS GATES
pod-01   1/1     Running   0          46s   20.110.126.16   k8s-worker2   <none>           <none>

$ curl 20.110.126.16:8080/hostname
pod-01
```  

#### 1-5. service로 접속확인
```shell
$ kubectl get svc -n nm-01 -o wide
NAME     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE    SELECTOR
svc-01   ClusterIP   10.101.253.125   <none>        9000/TCP   3m7s   app=pod

$ curl 10.101.253.125:9000/hostname
pod-01
```  
nm-01 Namespace에 포함된 Pod, Service의 연결을 확인해 보았다.  

### 2. Namespace 예외 - Service 생성(NodePort)
Service 오브젝트는 Namespace별로 관리된다고 했는데 Service의 Type이 NodePort인 경우는 예외이다. 아래 yaml 파일을 nm-01에서 실행 후 nm-02에서 또 실행을 하게 되면 이미 존재하는 서비스라는 오류 메세지가 출력된다.  
```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: nm-01
  #namespace: nm-02
  name: svc-02
spec:
  ports:
  - port: 9000
    targetPort: 8080
    nodePort: 30000
  type: NodePort
```  
### 3. Namespace 예외 - Pod 생성(HostPath)  
nm-02 Namespace 생성하고 nm-01,nm-02 Namespace에 pod-02 Pod를 생성한다. Namespace nm-01/pod-01 Container에서 생성된 파일을 nm-02/pod-01 Container에서도

#### 3-1. nm-02 namespace 생성
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nm-02
EOF
```  

#### 3-2. Pod pod-02를 nm-01,nm-02 Namespace에 생성한다.
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  #namespace: nm-01
  namespace: nm-02
  name: pod-02
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1
  containers:
  - name: container
    image: coolguy239/init
    volumeMounts:
    - name: host-path
      mountPath: /mount01
  volumes:
  - name : host-path
    hostPath:
      path: /node-mount01
      type: DirectoryOrCreate
EOF
```  

#### 3-3. nm-01/pod-02 Pod의 Container에서 HostPath 경로에 파일 생성
```shell
$ kubectl exec -ti pod-02 -n nm-01 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

$ cd ./mount01/
$ echo "test" >> test.txt
```  

#### 3-4. nm-02/pod-02 Pod의 Container에서 HostPath 경로에 파일 확인
```shell
$ kubectl exec -ti pod-02 -n nm-02 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

$ cat ./mount01/test.txt
test
```

서로 다른 Namespace의 HostPath로 설정한 Volume 경로의 파일은 기본적으로 공유된다.  

## ResourceQuota
Namespace 마다 설정이 가능하고 최대 허용 자원(CPU,Memory)을 설정한다. ResourceQuata가 지정된 Namespace에 Pod를 생성할 경우에는 반드시 requests,limits Spec을 명시해야 한다. ResourceQuata로 CPU,Memory,Storage,일부 오브젝트들의 숫자를 제한할 수 있다.  

## LimitRange
Namespace에 들어오는 Pod의 자원의 범위를 설정한다. maxLimitRequestRatio 값으로 limit와 request의 비율설정도 가능하다.  


## 참고
[KUBETM BLOG](https://kubetm.github.io/k8s/)   
[쿠버네티스 공식사이트](https://kubernetes.io/ko/docs/home/)
