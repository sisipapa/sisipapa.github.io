---
layout: post
title: CKA 자격 시험 준비를 위한 Kubernetes 정리2-Volume,ConfigMap,Namespace 
category: [k8s]
tags: [k8s, cka, kubernetes]
redirect_from:

- /2021/09/04/

---

오늘은 인프런 강의 대세는 쿠버네티스 강의 중 Volume, ConfigMap, Namespace 관련 내용을 실습해 보면서 정리 할 예정이다.  
Volume - <https://kubetm.github.io/k8s/03-beginner-basic-resource/volume/>   
ConfigMap - <https://kubetm.github.io/k8s/03-beginner-basic-resource/ConfigMap/>   
Namespace - <https://kubetm.github.io/k8s/03-beginner-basic-resource/Namespace/>    

## Volume  
### 1. emptyDir  
2개의 Container가 있는 Pod를 생성하고 2개의 Container 내에서 파일을 공유하는지 확인. emptyDir volume은 pod가 삭제가 되면 volume내의 파일도 같이 삭제된다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-01
spec:
  containers:
  - name: container01
    image: coolguy239/p8080
    volumeMounts:
    - name: empty-dir
      mountPath: /mount1
  - name: container02
    image: coolguy239/p8000
    volumeMounts:
    - name: empty-dir
      mountPath: /mount2
  volumes:
  - name : empty-dir
    emptyDir: {}
EOF
```  

container1 내부로 들어가서 mount1 경로가 생성된 것을 확인 후 mount1 경로에 test.txt 파일을 생성한다.
```shell
$ kubectl exec -ti -c container01 pod-vol-01 sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

# ls
app.jar  bin      dev      etc      home     lib      lib64    media    mnt      mount1   opt      proc     root     run      sbin     srv      sys      tmp      usr      var
# echo "this is test file" >> test.txt
```  
container2 내부로 들어가서 mount2 경로가 생성된 것을 확인 후 mount2 경로에 test.txt 파일의 내용을 확인한다.  
```shell
$ kubectl exec -ti -c container02 pod-vol-01 sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

# ls
app.jar  bin      dev      etc      home     lib      lib64    media    mnt      mount2   opt      proc     root     run      sbin     srv      sys      tmp      usr      var
# cat /mount2/test.txt
this is test file
```  

### 2. hostPath  
spec.volumes.hostPath.type = DirectoryOrCreate 의 의미는 Node에 해당 경로가 없으면 생성하겠다는 의미이다.   
hostPath 테스트 진행을 위해 k8s-worker1 노드에 동일한 Pod 두개를 생성한다.  

```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-02
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1
  containers:
  - name: container
    image: coolguy239/init
    volumeMounts:
    - name: host-path
      mountPath: /mount1
  volumes:
  - name : host-path
    hostPath:
      path: /node-v
      type: DirectoryOrCreate
EOF

--

$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-03
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-worker1
  containers:
  - name: container
    image: coolguy239/init
    volumeMounts:
    - name: host-path
      mountPath: /mount1
  volumes:
  - name : host-path
    hostPath:
      path: /node-v
      type: DirectoryOrCreate
EOF
```  

pod-vol-02 파드에 접속 후 mount1 경로에 test.txt 파일 생성
```shell
$ kubectl exec -ti pod-vol-02 sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
# ls
app.jar  bin  boot  dev  etc  home  lib  lib64	media  mnt  mount1  opt  proc  root  run  sbin	srv  sys  tmp  usr  var
# cd mount1
# echo "this is test file" >> test.txt
```  

pod-vol-03 파드에 접속 후 mount1 경로에 test.txt 파일 확인
```shell
$ kubectl exec -ti pod-vol-02 sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
# ls
app.jar  bin  boot  dev  etc  home  lib  lib64	media  mnt  mount1  opt  proc  root  run  sbin	srv  sys  tmp  usr  var
# cd mount1
# cat test.txt
this is test file
```  
위의 결과로 hostPath를 통해 같은 Node 내의 Pod들이 Volume을 공유하는 것을 확인 할 수 있다.  

k8s-worker1 노드의 /node-v 경로에 test.txt 파일 확인  
```shell
$ ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  node-v  opt  proc  root  run  sbin  srv  swapfile  sys  tmp  usr  vagrant  var
$ cat node-v/test.txt
this is test file
```  
hostPath로 잡은 volume은 Node에 Directory가 생기는 것이기 때문에 Pod가 삭제되도 Volume의 파일이 삭제되지 않는다. 

### 3. PVC / PV  - storage, accessModes로 연결
Pod의 Volume을 영속적으로 사용하기 위한 방법이다.

#### 3-1. PersistentVolume  
PVC와 PV가 capacity, accessModes로 연결되는 예제로 capacity, accessModes를 조금씩 다르게 설정한다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-01
spec:
  capacity:
    storage: 1G
  accessModes:
  - ReadWriteOnce
  local:
    path: /node-v
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - {key: kubernetes.io/hostname, operator: In, values: [k8s-worker1]}
EOF

--

$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-02
spec:
  capacity:
    storage: 1G
  accessModes:
  - ReadOnlyMany
  local:
    path: /node-v
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - {key: kubernetes.io/hostname, operator: In, values: [k8s-worker1]}
EOF

--

$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-03
spec:
  capacity:
    storage: 2G
  accessModes:
  - ReadWriteOnce
  local:
    path: /node-v
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - {key: kubernetes.io/hostname, operator: In, values: [k8s-worker1]}
EOF
```  

#### 3-2. PersistentVolumeClaim  
PVC 설정을 ReadWriteOnce, 1G로 설정하고 PVC를 생성하고 PV의 조회 결과를 보면 pv-01(1G, RWO)에 연결된 것을 확인 할 수 있다. PV보다 큰 PVC의 storage는 PV에 연결이 안되지만 작은 PVC는 accessModes만 일치한다면 연결이 가능하다.    
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-01
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1G
  storageClassName: ""
EOF

$  kubectl get pv -o wide
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM            STORAGECLASS   REASON   AGE     VOLUMEMODE
pv-01   1G         RWO            Retain           Bound       default/pvc-01                           5m35s   Filesystem
pv-02   1G         ROX            Retain           Available                                            4m54s   Filesystem
pv-03   2G         RWO            Retain           Available                                            2m58s   Filesystem

---

$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-02
spec:
  accessModes:
  - ReadOnlyMany
  resources:
    requests:
      storage: 1G
  storageClassName: ""
EOF

$  kubectl get pv -o wide
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM            STORAGECLASS   REASON   AGE     VOLUMEMODE
pv-01   1G         RWO            Retain           Bound       default/pvc-01                           9m32s   Filesystem
pv-02   1G         ROX            Retain           Bound       default/pvc-02                           8m51s   Filesystem
pv-03   2G         RWO            Retain           Available                                            6m55s   Filesystem

---

$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-03
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1G
  storageClassName: ""
EOF

$  kubectl get pv -o wide
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE   VOLUMEMODE
pv-01   1G         RWO            Retain           Bound    default/pvc-01                           14m   Filesystem
pv-02   1G         ROX            Retain           Bound    default/pvc-02                           14m   Filesystem
pv-03   2G         RWO            Retain           Bound    default/pvc-03                           12m   Filesystem
```  

#### 3-3. Pod  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-04
spec:
  containers:
  - name: container
    image: coolguy239/init
    volumeMounts:
    - name: pvc-pv
      mountPath: /mount3
  volumes:
  - name : pvc-pv
    persistentVolumeClaim:
      claimName: pvc-01
EOF

$ kubectl exec -ti pod-vol-04 sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
# ls
app.jar  bin  boot  dev  etc  home  lib  lib64	media  mnt  mount3  opt  proc  root  run  sbin	srv  sys  tmp  usr  var
```  

### 4. PVC / PV  - label과 selector를 이용한 연결  
이번 예제는 단순히 PVC와 PV만을 연결하기 위한 예제로 pod는 따로 생성하지 않는다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-04
  labels:
    pv: pv-04
spec:
  capacity:
    storage: 1G
  accessModes:
  - ReadWriteOnce
  local:
    path: /node-v
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - {key: kubernetes.io/hostname, operator: In, values: [k8s-worker1]}
EOF

---

$ kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-04
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1G
  storageClassName: ""
  selector:
    matchLabels:
      pv: pv-04
EOF

---

$ kubectl get pv -o wide
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE    VOLUMEMODE
pv-01   1G         RWO            Retain           Bound    default/pvc-01                           27m    Filesystem
pv-02   1G         ROX            Retain           Bound    default/pvc-02                           26m    Filesystem
pv-03   2G         RWO            Retain           Bound    default/pvc-03                           24m    Filesystem
pv-04   1G         RWO            Retain           Bound    default/pvc-04                           106s   Filesystem

```

## ConfingMap, Secret  
### 1. Literal 방식  
#### 1-1. ConfigMap
ConfigMap 생성시 key,value 형태로 yaml을 생성한다. Boolean형은 싱글쿼테이션으로 감싸준다.  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-01
data:
  data1: 'false'
  data2: stringValue
EOF
```  

#### 1-2. Secret
Secret은 yaml파일 생성시 value 값을 base64 Encoding을 해서 생성한다.  
```shell
$ echo -n '1111' | base64
MTExMQ==

$ echo -n '2222' | base64
MjIyMg==
```  
```shell 
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: sec-01
data:
  pwd1: MTExMQ==
  pwd2: MjIyMg==
EOF
```  

#### 1-3. Pod
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-01
spec:
  containers:
  - name: container
    image: coolguy239/init
    envFrom:
    - configMapRef:
        name: cm-01
    - secretRef:
        name: sec-01
EOF
```  

#### 1-4. Env 확인
pod-1 Container로 접속해서 env 환경변수를 확인한다.
```shell
$ kubectl exec -ti pod-01 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
$ env
...
pwd2=2222
pwd1=1111
...
data2=stringValue
...
daga1=false
...
```  
### 2. File 방식  
> --form-file={{file name}} : 파일 이름과 동일한 시크릿 데이터 키를 사용해 파일에서 적재    
> --from-file={{키}}={{file name}} : 명시적으로 지정된 시크릿 데이터 키를 사용해 파일에서 적재    
> --form-file={{디렉토리}} : 파일이름이 수용할 수 있는 키 이름이 지정된 디렉토리의 모든 파일을 적재    
> --form-literal={{키}={{값}} : 지정된 키/값 쌍으로 직접 사용  

#### 2-1. file로 ConfigMap, Secret 생성  
```shell
$ echo "val" >> data.txt
$ kubectl create configmap cm-02 --from-file=./cm-data.txt

$ echo "3333" >> data.txt
$ kubectl create secret generic sec-02 --from-file=./sec-data.txt
```  
#### 2-2. Pod  
```shell
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-02
spec:
  containers:
  - name: container
    image: coolguy239/init
    env:
    - name: cm-02
      valueFrom:
        configMapKeyRef:
          name: cm-02
          key: cm-data.txt
    - name: sec-02
      valueFrom:
        secretKeyRef:
          name: sec-02
          key: sec-data.txt
EOF
```  



## Namespace

## 참고
[KUBETM BLOG](https://kubernetes.io/ko/docs/concepts/workloads/pods/)  
[쿠버네티스 공식사이트](https://kubetm.github.io/k8s/)   
[WEBNORI - Kubernetes](http://wiki.webnori.com/pages/viewpage.action?pageId=12583285)   
