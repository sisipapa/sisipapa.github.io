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

## 1.  Replicas  

## 2. Updating Controller
## 3. Selector


## 참고  
[KUBETM BLOG](https://kubetm.github.io/k8s/04-beginner-controller/replicaset/)     
[쿠버네티스 공식사이트-ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)  
[쿠버네티스 공식사이트-ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)  
