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

## 참고  
[kubernetes 인증서 만료](https://kangwoo.github.io/devops/kubernetes/apiserver-kubelet-client-certs-expired/)  

