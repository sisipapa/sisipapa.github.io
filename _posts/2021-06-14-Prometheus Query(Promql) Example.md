---
layout: post 
title: Prometheus Query(Promql) Example
category: [prometheus]
tags: [prometheus, promql]
redirect_from:

- /2021/06/14/

---
[k8s+Prometheus+Grafana 연동](https://sisipapa.github.io/blog/2021/01/01/k8s-prometheus-grafana/)  
개인 스터디를 하면서 k8s+Prometheus+Grafana 설정을 정리했었는데 최근 운영 중인 플랫폼의 쿠버네티스 장애가 발생하면서 팀 내부적으로 모니터링을 위한 대시보드 구성을 해보자는 의견이 나오게 되었고 Prometheus 서버와 node-exporter를 설치를 한 상태이다. Grafana 화면구성을 위해 Promql의 결과를 보면서 Qeury Exampled를 정리해 보려고 한다.

#### 모든 데이터 조회
```shell
kubelet_http_requests_total
```  

#### 한개 필드 equal 조회
```shell
kubelet_http_requests_total{method="GET"}
```  

#### 다중 필드 equal 조회
```shell
kubelet_http_requests_total{beta_kubernetes_io_os="linux",method="GET"}
```  

#### 필드 like 조회  - instance cluster 포함 검색
```shell
kubelet_http_requests_total{instance=~".*cluster.*"}
```  








    
  



 

