---
title: kubernetes 테스트 환경 설치 가이드 (2)
description: kubernetes 테스트 환경 설치 가이드 (2)
author: yoonxjoong
date: 2025-03-11 10:00:00 +0900
categories:
  - DevOps
tags:
  - Kubernates
  - Kubeadm
  - Kubelet
  - Kubectl
---
## 쿠버네티스 설치 가이드 (노드 서버)

- Mac에서 UTM 가상머신을 통해 마스터 서버, 노드 서버 2개를 생성하여 쿠버네티스 테스트 환경을 구축하면서 정리한 문서입니다. 


#### 1. 네트워크 기본 설정
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



#### 3. kubelet,  kubectl 설치 (kubeadm는 설치하지 않음)
- 설치 스크립트 작성 (install_kubernetes_node.sh)

```shell

#!/bin/bash
set -e

echo "============================"
echo " 기존 Kubernetes 설정 초기화"
echo "============================"
sudo kubeadm reset -f || true
sudo systemctl stop kubelet
sudo rm -rf /etc/kubernetes /var/lib/kubelet /etc/cni /opt/cni

echo "============================"
echo " 스왑 비활성화"
echo "============================"
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

echo "============================"
echo " 네트워크 설정 (IP 포워딩 및 브릿지 설정)"
echo "============================"
sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
EOF
sudo sysctl --system

echo "============================"
echo " Containerd 설치 및 설정"
echo "============================"
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable --now containerd

echo "============================"
echo " Kubernetes 저장소 설정"
echo "============================"
sudo rm -f /etc/apt/sources.list.d/kubernetes.list
sudo rm -f /etc/apt/keyrings/kubernetes-archive-keyring.gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

echo "============================"
echo " 패키지 목록 업데이트 및 Kubernetes 패키지 설치"
echo "============================"
sudo apt-get update
sudo apt-get install -y kubelet kubeadm
sudo systemctl enable --now kubelet

echo "============================"
echo " Kubernetes 워커(노드) 설치 완료!"
echo "============================"

# 인자가 3개이면 자동으로 클러스터에 조인, 그렇지 않으면 조인 명령어 안내 출력
if [ "$#" -eq 3 ]; then
    MASTER_IP="$1"
    TOKEN="$2"
    DISCOVERY_HASH="$3"
    echo "클러스터에 자동으로 조인합니다:"
    echo "  sudo kubeadm join ${MASTER_IP}:6443 --token ${TOKEN} --discovery-token-ca-cert-hash sha256:${DISCOVERY_HASH}"
    sudo kubeadm join ${MASTER_IP}:6443 --token ${TOKEN} --discovery-token-ca-cert-hash sha256:${DISCOVERY_HASH}
else
    echo "마스터 노드에서 생성된 'kubeadm join' 명령어를 사용하여 클러스터에 조인하세요."
    echo "예:"
    echo "  sudo kubeadm join 192.168.64.100:6443 --token <your-token> --discovery-token-ca-cert-hash sha256:<your-hash>"
fi

```


- 스크립트 실행 권한 부여
``` shell
chmod +x install_kubernetes_node.sh
```

- 스크립트 실행
``` shell
./install_kubernetes_node.sh
```
