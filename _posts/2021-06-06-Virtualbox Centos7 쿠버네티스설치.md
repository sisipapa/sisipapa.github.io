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
예전에 학습하다가 마무리하지 못한 인프런 강의를 공부를 위해 강의와 동일한 환경을 구축할 예정이다. (Window + Virtualbox(Network:NatNetwork))  
  
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

2. VM 설정  
2-1. 머신 > 새로 만들기 클릭
2-2. 이름 : k8s-master, 종류: Linux, 버전: Other Linux(64-bit)  
   <img src="https://sisipapa.github.io/assets/images/posts/vm1.PNG" >        
2-3. 메모리 : 4096 MB  
   <img src="https://sisipapa.github.io/assets/images/posts/vm2.PNG" >  
2-4. 하드디스크 : 지금 새 가상 하드 디스크 만들기 (VDI:VirtualBox 디크스 이미지, 동적할당, 150GB)
   <img src="https://sisipapa.github.io/assets/images/posts/vm3.PNG" >  
   <img src="https://sisipapa.github.io/assets/images/posts/vm4.PNG" >  
   <img src="https://sisipapa.github.io/assets/images/posts/vm5.PNG" >  
   <img src="https://sisipapa.github.io/assets/images/posts/vm6.PNG" >  
2-5. 설정 클릭  
- [시스템] 프로세서 개수 : CPU 2개
  <img src="https://sisipapa.github.io/assets/images/posts/vm-set1.PNG" >  
  
- [저장소] 컨트롤러:IDE 하위에 있는 광학드라이브 클릭 > CentOS 이미지 선택 후 확인  
  <img src="https://sisipapa.github.io/assets/images/posts/vm-set2.PNG" >  
  
- [네트워크] 네트워크 > 어댑터 1 탭 > 다음에 연결됨 [NAT 네트워크] 선택  
  <img src="https://sisipapa.github.io/assets/images/posts/vm-set3.PNG" >  
  
  
## CentOS 설치   
1. Language : 한국어  
   <img src="https://sisipapa.github.io/assets/images/posts/centos1.PNG" >  
2. Disk 설정 시스템 > 설치대상 클릭한다.  
2-1. 기타 저장소 옵션 > 파티션 설정 파티션을 설정합니다. 선택하고 완료 버튼을 클릭한다.  
   <img src="https://sisipapa.github.io/assets/images/posts/centos2.PNG" >    
2-2. 새로운 CentOS 설치 > 여기를 클릭하여 자동으로 생성합니다. 링크를 클릭한다.  
   <img src="https://sisipapa.github.io/assets/images/posts/centos3.PNG" >  
2-3. /home 용량 5.12 GiB로 변경한다.  
2-4. /  140 GiB 변경한다.   
   <img src="https://sisipapa.github.io/assets/images/posts/centos4.PNG" >  
3. 네트워크 설정  
3-1. 호스트 이름 변경 후 적용 버튼 클릭 후 설정 버튼 클릭  
   <img src="https://sisipapa.github.io/assets/images/posts/centos5.PNG" >  
3-2. IPv4설정 탭 > 방식: 수동으로 선택  
3-3. Add클릭 > 주소: 30.0.2.30, 넷마스크 : 255.255.255.0, 게이트웨이: 30.0.2.1, DNS 서버 : 8.8.8.8  
   <img src="https://sisipapa.github.io/assets/images/posts/centos6.PNG" >  
4. 설정 > 사용자 설정 ROOT 암호 설정   

## 도커 / 쿠버네티스 설치 전 설정사항
설정항목들의 설명은 아래 참고 블로그의 내용을 참고하면 좋을 것 같다.  

1. SELinux 설정   

```shell  
setenforce 0  

sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config  

sestatus  
```   


2. 방화벽 해제  

```shell  
systemctl stop firewalld && systemctl disable firewalld  

systemctl stop NetworkManager && systemctl disable NetworkManager
```   


3. Swap 비활성화   

```shell
swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab  
```   


4. Iptables 커널 옵션 활성화  
```shell
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```  

5. 쿠버네티스 YUM Repository 설정  
```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```  

6. Centos Update  
```shell
yum -y update
```  

7. Hosts 등록  
```shell
cat << EOF >> /etc/hosts
30.0.2.30 k8s-master
30.0.2.31 k8s-node1
30.0.2.32 k8s-node2
EOF
```  

## 도커, 쿠버네티스 설치   

1. 도커설치  

```shell  

yum install -y yum-utils device-mapper-persistent-data lvm2  

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo  

yum update -y && yum install -y containerd.io-1.2.13 docker-ce-19.03.11 docker-ce-cli-19.03.11

mkdir /etc/docker

cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d
```   


2. 쿠버네티스 설치  
```shell
yum install -y --disableexcludes=kubernetes kubeadm-1.19.4-0.x86_64 kubectl-1.19.4-0.x86_64 kubelet-1.19.4-0.x86_64
```  

## 버추얼박스 VM 복제  
1. 복제를 위해 master 노드를 shutdown 시킨다.
```shell
shutdown now
```  
2. 이름 : k8s-node1(k8s-node2), MAC 주소정책 : 모든 네트워크 어댑터의 새 MAC 주소 생성  
3. 복제방식 : 완전한 복제  
4. 복제 후 k8s-node1(k8s-node2) VM에 접속해서 hostname 변경  
```shell
hostnamectl set-hostname k8s-node1
```  
5. 복제 후 k8s-node1(k8s-node2) IPADDR 변경   

```shell  
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3   
  
TYPE="Ethernet"  
PROXY_METHOD="none"  
BROWSER_ONLY="no"  
BOOTPROTO="none"  
DEFROUTE="yes"  
IPV4_FAILURE_FATAL="no"  
IPV6INIT="yes"  
IPV6_AUTOCONF="yes"  
IPV6_DEFROUTE="yes"  
IPV6_FAILURE_FATAL="no"  
IPV6_ADDR_GEN_MODE="stable-privacy"  
NAME="enp0s3"  
UUID="b573f630-a22c-483d-bec9-f93b6d2bde49"  
DEVICE="enp0s3"  
ONBOOT="yes"  
IPADDR="30.0.2.31"  
PREFIX="24"  
GATEWAY="30.0.2.1"  
DNS1="8.8.8.8"  
IPV6_PRIVACY="no"  
```    


## Master 노드
1. 도커 및 쿠버네티스 실행  
```shell
systemctl daemon-reload  

systemctl enable --now docker

systemctl enable --now kubelet
```  
2. 쿠버네티스 초기화 명령실행  
```shell
kubeadm init --pod-network-cidr=20.96.0.0/12 --apiserver-advertise-address=30.0.2.30
```  
3. 2번의 실행 후 결과를 복사
```shell
...
kubeadm join 30.0.2.30:6443 --token 0kdc8w.hszdvr3hvz2ldd5j \
    --discovery-token-ca-cert-hash sha256:05eb3e67e477627580bfe4d5460e4760753b1cc5c163b392df9be026991bd300
... 
```  
4. 환경변수 설정  
```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```  
5. Kubectl 자동완성 기능 설치  
```shell
yum install bash-completion -y
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```  

## Work 노드  

1. 도커 및 쿠버네티스 실행   

```shell  
systemctl daemon-reload  

systemctl enable --now docker

systemctl enable --now kubelet  
```   


2. Master 노드의 연결 - Master 노드에서 실행 결과로 복사해 둔 내용 실행  

```shell  
kubeadm join 30.0.2.30:6443 --token 0kdc8w.hszdvr3hvz2ldd5j \
    --discovery-token-ca-cert-hash sha256:05eb3e67e477627580bfe4d5460e4760753b1cc5c163b392df9be026991bd300
```  

3. 연결확인  
```shell
kubectl get nodes
```

## 참고
[KubeTM Blog](https://kubetm.github.io/practice/appendix/installation_case5/)  
    
  



 

