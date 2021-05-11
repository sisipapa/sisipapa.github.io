---
layout: post 
title: Kubernetes 인증서 만료
category: [k8s]
tags: [k8s,인증서]
redirect_from:

- /2021/05/10/

---

## 인증서 만료일 확인
회사 쿠버네티스 인증서 만료일이 얼마 남지 않았다는 이야기를 들었다. 확인을 위해 인증서 만료일을 확인하기 위해서 마스터 노드에 접속 후 확인해 보았다. 확인을 해보니 인증서 만료일은 Jun 11 02:31:10 2021 GMT 이고 이 기간 내에 인증서 갱신을 하지 않으면 서비스에는 영향이 없지만 kubectl 명령을 정상적으로 사용할 수 없게 된다.      
  
#### 명령어로 직접 확인
```shell  
$ cd /etc/kubernetes/pki

$ openssl x509 -in apiserver.crt -noout -dates
notBefore=Dec  9 07:53:31 2019 GMT
notAfter=Jun 11 02:31:10 2021 GMT

$ openssl x509 -in apiserver-kubelet-client.crt -noout -dates
notBefore=Dec  9 07:53:31 2019 GMT
notAfter=Jun 11 02:31:10 2021 GMT

$ openssl x509 -in front-proxy-client.crt -noout -dates
notBefore=Dec  9 07:53:32 2019 GMT
notAfter=Jun 11 02:31:11 2021 GMT
```
  
#### check.sh 스크립트를 만들어서 확인
```shell  
$ cat check.sh 
for crt in /etc/kubernetes/pki/*.crt; do
     printf '%s: %s\n' \
     "$(date --date="$(openssl x509 -enddate -noout -in "$crt"|cut -d= -f 2)" --iso-8601)" \
     "$crt"
done | sort

$ sh check.sh
2021-06-11: /etc/kubernetes/pki/apiserver-kubelet-client.crt
2021-06-11: /etc/kubernetes/pki/apiserver.crt
2021-06-11: /etc/kubernetes/pki/front-proxy-client.crt
2029-12-06: /etc/kubernetes/pki/ca.crt
2029-12-06: /etc/kubernetes/pki/front-proxy-ca.crt
```  

## 인증서 생성
apiserver-kubelet-client 인증서만 다시 생성하면 되지만 만료일이 2021-06-11 로 같은 apiserver, front-proxy-client 인증서도 생성할 예정이다.  
#### 인증서 백업
```shell  
$ cd /etc/kubernetes/pki
$ mv apiserver.key apiserver.key.old
$ mv apiserver.crt apiserver.crt.old
$ mv apiserver-kubelet-client.key  apiserver-kubelet-client.key.old
$ mv apiserver-kubelet-client.crt  apiserver-kubelet-client.crt.old
$ mv apiserver-etcd-client.key front-proxy-client.key.old
$ mv apiserver-etcd-client.crt front-proxy-client.crt.old
```  
#### X509v3 Subject Alternative Name 확인  
X509v3 Subject Alternative Name 항목에 마스터 노드의 IP가 포함되어 있어야 한다.
```shell  
... 생략

X509v3 Subject Alternative Name: 
                DNS:xxx-cluster-01, DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc, DNS:kubernetes.default.svc.mobon.platform, IP Address:xx.xx.x.1, IP Address:xx.xxx.x.191, IP Address:xx.xxx.x.194
                
...생략
```
#### kubeadm을 이용한 인증서 생성
--apiserver-cert-extra-sans 파라미터의 값으로 X509v3 Subject Alternative Name에서 조회한 값을 넣고 실행할 예정. 결과는 해보고 업데이트 할 예정이다.
```shell  
$ kubeadm alpha phase certs apiserver --apiserver-cert-extra-sans 'X509v3 Subject Alternative Name에서 조회한 값'
$ kubeadm alpha phase certs apiserver-kubelet-client
$ kubeadm alpha phase certs front-proxy-client
```   

#### conf파일 다시 생성
```shell  
$ cd /etc/kubernetes
$ mv admin.conf admin.conf.old
$ mv kubelet.conf kubelet.conf.old
$ mv controller-manager.conf controller-manager.conf.old
$ mv scheduler.conf scheduler.conf.old

$ kubeadm alpha phase kubeconfig all
OR
$ kubeadm alpha phase kubeconfig admin
$ kubeadm alpha phase kubeconfig kubelet
$ kubeadm alpha phase kubeconfig controller-manager
$ kubeadm alpha phase kubeconfig scheduler
```  
인증서 업데이트를 하기 전 준비단계이다. 인증서 업데이트를 진행하고 결과를 반영할 예정이다.  

## 참고  
[kubernetes 인증서 만료](https://kangwoo.github.io/devops/kubernetes/apiserver-kubelet-client-certs-expired/)  
[K8S apiserver에 SAN(Subject Alternative Name) 추가](https://blusky10.tistory.com/498)  

