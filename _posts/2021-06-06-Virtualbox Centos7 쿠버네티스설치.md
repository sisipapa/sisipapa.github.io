---
layout: post 
title: Virtualbox Centos7 쿠버네티스설치
category: [k8s]
tags: [k8s, vmware, centos7]
redirect_from:

- /2021/06/06/

---

## 쿠버네티스 Virtualbox 설치를 위한 준비  
- [Virtualbox 다운로드](https://www.virtualbox.org/wiki/Downloads)  
- [Centos7 다운로드](http://isoredirect.centos.org/centos/7/isos/x86_64/)  
쿠버네티스 스터디를 진행하면서 Google 계정을 만들어서 GCP를 이용했었는데 최대 3개월 / 300달러 비용을 다 사용하고 나면 매번 새로운 Google 계정을 만들어서 GCP의 쿠버네티스 환경을 새로 구성해야했다. 그래서 로컬PC에 Virtualbox를 설치하고 쿠버네티스 환경을 구성해 보려고 한다.
예전에 학습하다가 중단한 인프런 강의를 공부할 겸 강의와 동일한 환경을 구축할 예정이다. (Window + Virtualbox(Network:NatNetwork))  
  
## Virtualbox 설정  
1. NatNetwork 생성  
```text
1. 파일 > 환경설정 > 네트워크 > 새 NAT 네트워크 생성 추가
2. 편집 버튼 클릭
   - 네트워크 CICR 수정 : 30.0.2.0/24
3. 포트 포워딩(P) 클릭 후 아래와 같이 설정
```  
<img src="https://sisipapa.github.io/assets/images/posts/버추얼박스1.PNG" >     
<img src="https://sisipapa.github.io/assets/images/posts/버추얼박스2.PNG" >     






## 참고
[KubeTM Blog](https://kubetm.github.io/practice/appendix/installation_case5/)  
    
  



 

