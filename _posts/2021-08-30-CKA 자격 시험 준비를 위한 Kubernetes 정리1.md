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
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
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
```  


## Service  
## Volume  
## ConfigMap  
## Namespace  

## 참고
[KUBETM BLOG](https://kubernetes.io/ko/docs/concepts/workloads/pods/)  
[쿠버네티스 공식사이트](https://kubetm.github.io/k8s/)   

## Github    