---
title: Kubernates - 신규 Deployment 생성 및 서비스 외부 노출
description: 신규 Deployment 생성하고 서비스 외부로 노출하는 yaml 을 작성하고 검증하는 내용
author: yoonxjoong
date: 2025-03-18 09:00:00 +0900
categories:
  - DevOps
tags:
  - Kubernates
  - Kubeadm
  - Kubelet
  - Kubectl
---
다음 조건을 충족하는 Kubernetes 리소스를 생성하십시오.

**Namespace:**  `exam-space`

**Deployment 이름:**  `web-app`

**Pod 구성:**

- 이미지: `nginx:1.23.3`
- 레플리카(replica) 수: `3`
- 리소스 제한(resources limit): CPU=`250m`, Memory=`128Mi`

**서비스 노출 조건:**

- 서비스 이름: `web-service`
- 서비스 유형: `NodePort`
- 포트: 컨테이너 포트 `80`, 서비스 포트 `8080`


``` bash
# 네임스페이스 생성
$ kubctl create namespace exam-space

# Pod 생성
$ kubectl deployment web-app --image=nginx:1.23.3 --replicas=3 --namespace=exam-space --dry-run=client -o yaml > web-app.yaml

# 리소스 제한 추가
$ vi web-app.yaml

"web-app.yaml"
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: web-app
  name: web-app
  namespace: exam-space
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web-app
    spec:
      containers:
      - image: nginx:1.23.3
        name: nginx
        resources:
          limits: # 리소스 제한 조건
            cpu: 250m # cpu
            memory: 120Mi # memory

status: {}

:wq

# 쿠버네티스 실행
$ kubectl apply -f web-app.yaml

# 리소스 검증
$ kubectl get all -n exam-space

NAME                           READY   STATUS    RESTARTS   AGE
pod/web-app-69c44dccf5-8v5mg   1/1     Running   0          4m47s
pod/web-app-69c44dccf5-fbk48   1/1     Running   0          4m47s
pod/web-app-69c44dccf5-nf6t2   1/1     Running   0          4m47s

NAME                  TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/web-service   NodePort   10.111.244.165   <none>        8080:32744/TCP   2m40s

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/web-app   3/3     3            3           4m47s

NAME                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/web-app-69c44dccf5   3         3         3       4m47s

# 서비스 노출
$ kubectl expose deployment web-app \
  --name=web-service \
  --namespace=exam-space \
  --type=NodePort \
  --port=8080 \
  --target-port=80

# 리소스 검증 2
$ kubectl describe service web-service -n exam-space

# 출력값
master@master:~$ kubectl describe service web-service -n exam-space
Name:                     web-service
Namespace:                exam-space
Labels:                   app=web-app
Annotations:              <none>
Selector:                 app=web-app
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.111.244.165
IPs:                      10.111.244.165
Port:                     <unset>  8080/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  32744/TCP
Endpoints:                192.168.166.142:80,192.168.166.143:80,192.168.166.144:80
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
