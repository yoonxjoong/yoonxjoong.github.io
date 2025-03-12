---
title: Kubernates Secret
description: Kubernates Secret를 생성하고 환경변수로 주입하는 실습
author: yoonxjoong
date: 2025-03-12 09:00:00 +0900
categories:
  - DevOps
tags:
  - Kubernates
  - Kubeadm
  - Kubelet
  - Kubectl
---
이 문제는 Kubernetes Secret을 생성하고, 이를 Pod의 환경 변수로 주입하는 작업을 통해 민감한 데이터를 안전하게 관리하는 방법을 실습하는 내용입니다.

``` yaml
---
# Namespace 생성
apiVersion: v1
kind: Namespace
metadata:
  name: secretdemo
---
# Secret 생성
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials # Secret 리소스 생성
  namespace: secretdemo
type: Opaque # 키 - 값 쌍 데이터를 저장하기 위해 Opaque 타입을 사용
data:
  username: YWRtaW4=
  password: c2VjcmV0cGFzc3dvcmQ=
---
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod # Pod 리소스 생성
  namespace: secretdemo
spec:
  containers:
  - name: busybox
    image: busybox # 가벼운 리눅스 환경을 구성
    command: ["sh", "-c", "echo Username: $DB_USERNAME && echo Password: $DB_PASSWORD && sleep 3600"]
    env:
    - name: DB_USERNAME
      valueFrom: # db-credentials에서 해당키를 가져옴
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
~
```

