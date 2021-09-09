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
### 1. Namespace ResourceQuota 설정 후 Pod 생성 테스트
#### 1-1. Namespace
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nm-03
EOF
```  

#### 1-2. ResourceQuota
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-01
  namespace: nm-03
spec:
  hard:
    requests.memory: 1Gi
    limits.memory: 1Gi
EOF


$ kubectl describe resourcequotas -n=nm-03
Name:            rq-01
Namespace:       nm-03
Resource         Used  Hard
--------         ----  ----
limits.memory    0     1Gi
requests.memory  0     1Gi

```  

#### 1-3. Pod 생성 오류
spec.containers.resources.request,spec.containers.resources.limits 설정안한 Pod는 아래와 같은 오류가 난다.
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
  namespace: nm-03
spec:
  containers:
  - name: container
    image: coolguy239/app
EOF
Error from server (Forbidden): error when creating "STDIN": pods "pod-01" is forbidden: failed quota: rq-01: must specify limits.memory,requests.memory
```  

#### 1-4. Pod 생성 정상
spec.containers.resources.request,spec.containers.resources.limits 설정 한 Pod는 정상 create 확인
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-02
  namespace: nm-03
spec:
  containers:
  - name: container
    image: coolguy239/app
    resources:
      requests:
        memory: 0.5Gi
      limits:
        memory: 0.5Gi
EOF
pod/pod-02 created
```  

#### 1-4. Pod 생성 오류
pod-02를 requests.memory: 0.5Gi, limits.memory: 0.5Gi로 생성했기 때문에 남는 자원보다 큰 reqquests,limits 설정 시 오류.
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-03
  namespace: nm-03
spec:
  containers:
  - name: container
    image: coolguy239/app
    resources:
      requests:
        memory: 0.7Gi
      limits:
        memory: 0.7Gi
EOF
Warning: spec.containers[0].resources.limits[memory]: fractional byte value "751619276800m" is invalid, must be an integer
Warning: spec.containers[0].resources.requests[memory]: fractional byte value "751619276800m" is invalid, must be an integer
Error from server (Forbidden): error when creating "STDIN": pods "pod-03" is forbidden: exceeded quota: rq-01, requested: limits.memory=751619276800m,requests.memory=751619276800m, used: limits.memory=512Mi,requests.memory=512Mi, limited: limits.memory=1Gi,requests.memory=1Gi
```  

### 2. Namespace ResourceQuota Pod 갯수 제한
#### 2-1. Namespace 생성
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nm-04
EOF
```

#### 2-2. ResourceQuata 생성
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ResourceQuota
metadata:
  name: rq-02
  namespace: nm-04
spec:
  hard:
    pods: 2
EOF
```  

#### 2-3. Pod생성 - metadata.name만 바꿔서 세번실행
pod-01,pod-02는 정상적으로 생성되었지만 pod-03은 ResourceQuota의 pod수 제한에 걸려 생성 오류 발생함.
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  #name: pod-01
  #name: pod-02
  name: pod-03
  namespace: nm-04
spec:
  containers:
  - name: container
    image: coolguy239/app
EOF
Error from server (Forbidden): error when creating "STDIN": pods "pod-03" is forbidden: exceeded quota: rq-02, requested: pods=1, used: pods=2, limited: pods=2
```

## LimitRange
Namespace에 들어오는 Pod의 자원의 범위를 설정한다. maxLimitRequestRatio 값으로 limit와 request의 비율설정도 가능하다.  
ResourceQuota는 Namespace 뿐만 아니라 Cluster 전체에 부여할 수 있는 권한이지만, LimitRange의 경우 Namespace내에서만 사용 가능하다.  
### 1. LimitRange에 설정한 
#### 1-1. Namespace
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nm-05
EOF
```  

#### 1-2. LimitRange
type: Container => Container마다 제한을 걸겠다는 의미.
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: lr-01
  namespace: nm-05
spec:
  limits:
  - type: Container # Container마다 제한을 걸겠다는 의미.
    min:
      memory: 0.1Gi
    max:
      memory: 0.4Gi
    maxLimitRequestRatio:
      memory: 3
    defaultRequest:
      memory: 0.1Gi
    default:
      memory: 0.2Gi
EOF

$ kubectl describe limitranges -n=nm-05
Name:       lr-01
Namespace:  nm-05
Type        Resource  Min            Max            Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---            ---            ---------------  -------------  -----------------------
Container   memory    107374182400m  429496729600m  107374182400m    214748364800m  3
```  

#### 1-3. Pod
maxLimitRequestRatio 비율이 3보다 높고, limits 메모리가 LimitRange에 설정한 0.4보다 크기때문에 Pod 생성시 오류가 난다. 
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
  namespace: nm-05
spec:
  containers:
  - name: container
    image: coolguy239/app
    resources:
      requests:
        memory: 0.1Gi
      limits:
        memory: 0.5Gi
EOF
Warning: spec.containers[0].resources.requests[memory]: fractional byte value "107374182400m" is invalid, must be an integer
Error from server (Forbidden): error when creating "STDIN": pods "pod-01" is forbidden: [maximum memory usage per Container is 429496729600m, but limit is 512Mi, memory max limit to request ratio per Container is 3, but provided ratio is 5.000000]
```    

#### 1-4. Pod 생성 default
Pod 생성 시 resources를 지정하지 않으면 LimitRange에 설정한 default 값으로 생성된다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-02
  namespace: nm-05
spec:
  containers:
  - name: container
    image: coolguy239/app
EOF
Warning: spec.containers[0].resources.limits[memory]: fractional byte value "214748364800m" is invalid, must be an integer
Warning: spec.containers[0].resources.requests[memory]: fractional byte value "107374182400m" is invalid, must be an integer
pod/pod-02 created

$ kubectl describe pod pod-02 -n nm-05
Name:         pod-02
Namespace:    nm-05
Priority:     0
Node:         k8s-worker2/192.168.56.32
Start Time:   Thu, 09 Sep 2021 14:41:26 +0000
Labels:       <none>
Annotations:  cni.projectcalico.org/containerID: 12920f6ad3027d337b748af10363289eedb10ec6ddbf48009f034fb03990ec80
              cni.projectcalico.org/podIP: 20.110.126.10/32
              cni.projectcalico.org/podIPs: 20.110.126.10/32
              kubernetes.io/limit-ranger: LimitRanger plugin set: memory request for container container; memory limit for container container
Status:       Running
IP:           20.110.126.10
IPs:
  IP:  20.110.126.10
Containers:
  container:
    Container ID:   docker://10350612dc81e6e18fad748ad054062341a07293de9b159381a35a114e91d5ec
    Image:          coolguy239/app
    Image ID:       docker-pullable://coolguy239/app@sha256:4749c0be4b64eaed9829e103ebc37c56e88627b88f179fe7f56cc07a9ef040f3
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 09 Sep 2021 14:41:30 +0000
    Ready:          True
    Restart Count:  0
    Limits:
      memory:  214748364800m
    Requests:
      memory:     107374182400m
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-47xlc (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-47xlc:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  23s        default-scheduler  Successfully assigned nm-05/pod-02 to k8s-worker2
  Normal  Pulling    <invalid>  kubelet            Pulling image "coolguy239/app"
  Normal  Pulled     <invalid>  kubelet            Successfully pulled image "coolguy239/app" in 2.615529079s
  Normal  Created    <invalid>  kubelet            Created container container
  Normal  Started    <invalid>  kubelet            Started container container
```  

## 참고  
[KUBETM BLOG](https://kubetm.github.io/k8s/)     
[쿠버네티스 공식사이트](https://kubernetes.io/ko/docs/home/)  
