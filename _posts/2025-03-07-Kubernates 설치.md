---
title: Kubernates 테스트 환경 설치 가이드
description: Kubernates 테스트 환경 설치 가이드
author: yoonxjoong
date: 2025-03-07 10:00:00 +0900
categories:
  - DevOps
tags:
  - Kubernates
  - Kubeadm
  - Kubelet
  - Kubectl
---
## 쿠버네티스 설치 가이드

- Mac에서 UTM 가상머신을 통해 마스터 서버, 노드 서버 2개를 생성하여 쿠버네티스 테스트 환경을 구축하면서 정리한 문서입니다. 


#### 1. 네트워크 기본 설정
- 마스터 아이피 주소 : 192.168.64.100
- 노드 1 아이피 주소 : 192.168.64.101
- 노드 2 아이피 주소 : 192.168.64.102

##### 네트워크 변경 명령어
- Netpaln 설정파일 열기
``` bash
sudo vi /etc/netplan/50-cloud-init.yaml
```

-  yaml 파일 수정
``` yaml
network:
	version: 2
	ethernets:
		enp0s1:
			addresses:
				- 192.168.64.100/24
			routes:
				- to: default
			 	  via: 192.168.64.1
			nameservers:
				addresses:
					- 8.8.8.8
					- 1.1.1.1
			dhcp4: no
```

- 변경 사항 적용
``` bash
sudo netplan apply
```

#### 2. SSH 접근 허용
- SSH 서버 설치
``` bash
sudo apt update && sudo apt install -y openssh-server
```

- SSH 서비스 시작 및 자동 실행
``` bash
sudo systemctl enable --now ssh
```

- SSH 상태 확인
``` bash
systemct status ssh
```



#### 3. kubelet,  kubectl, kubeadm 설치
- 설치 스크립트 작성 (install_kubernetes.sh)

```shell

#!/bin/bash
  
set -e  # 오류 발생 시 스크립트 중단  
  
echo "🔹 기존 Kubernetes 저장소 및 GPG 키 삭제..."  
sudo rm -f /etc/apt/sources.list.d/kubernetes.list  
sudo rm -f /etc/apt/keyrings/kubernetes-archive-keyring.gpg  
  
echo "🔹 최신 Kubernetes GPG 키 추가..."  
sudo mkdir -p /etc/apt/keyrings  
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg  
  
echo "🔹 Kubernetes 저장소 추가..."  
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list  
  
echo "🔹 패키지 목록 업데이트..."  
sudo apt-get update  
  
echo "🔹 Kubernetes 패키지 설치 (kubectl, kubelet, kubeadm)..."  
sudo apt-get install -y kubelet kubeadm kubectl  
  
echo "🔹 kubelet 서비스 활성화..."  
sudo systemctl enable --now kubelet  
  
echo "🔹 설치된 Kubernetes 버전 확인..."  
kubectl version --client  
kubeadm version  
kubelet --version  

echo "✅ Kubernetes 설치 완료!"

```


- 스크립트 실행 권한 부여
``` shell
chmod +x install_kubernetes.sh
```

- 스크립트 실행
``` shell
./install_kubernetes.sh
```
