---
title: Git Branch 전략
description: 업무 중 히스토리 분석이나 깔끔한 그래프를 위하여 Git branch 전략을 수정하면서 정리한 내용을 포스팅합니다.
author: yoonxjoong
date: 2025-02-27 10:00:00 +0900
categories:
  - DevOps
tags:
  - Git
  - Branch
  - 깃그래프
---
# Git Branch 전략
### 기존 Git Branch 규칙 
- 개발 담당자들은 Feature 브랜치를 따서 개발 내용을 커밋하고 필요에 따라 푸시를 합니다.
- 개발이 완료되거나 테스트가 필요한 소스는 Dev브랜치에 머지하고 테스트를 진행합니다. 
- 테스트가 완료되면 Release 브랜치에 머지를 하고 배포 담당자가 Prod브랜치에 최종 머지합니다. 
- 이때 배포하기전에 Prod 브랜치에서 Release 브랜치를 생성해야됩니다. (여러 개발자들이 Release 브랜치에 개발한 내용을 머지)
### 문제점?
- 깃 그래프의 히스토리가 굉장히 복잡하고 파악하기 어려웠습니다. 

### 해결 방식
- 개발자들은 브랜치를 Dev 브랜치에 머지하기 전에 리베이스를 통해 소스의 동기화를 시켜줍니다. 
- 그 이후 Dev 브랜치에 머지하여 개발 서버를 배포합니다. 

위에 방식대로 하면 Dev 브랜치에 히스토리는 보기 깔끔해지고 이력대로 히스토리를  파악하기 좋아졌습니다. 
여기서 또다른 문제가 발생합니다. 

### 또 다른 문제..
**Git Branch 규칙**에서도 설명 했듯이 저희 개발환경은 개발 서버에 테스트가 완료되면 Feature 브랜치를 Release에 머지를 합니다.

이때 Feature 브랜치는 Dev 브랜치을 리베이스 했으므로 다른 개발자가 테스트 용으로 올린 커밋들이 저장되어 있을거에요. 

그 다음에 Feature 브랜치를 Release 브랜치에 머지를 하게 된다면 다른 개발자들의 커밋들도 머지가 될것이고 누군가는 테스트로 올린 소스가 운영에 적용되어 큰 문제가 발생할 것입니다.

이를 해결하기 위해 Dev 브랜치의 히스토리는 깔끔하게 포기를 하고 Release 브랜치의 리베이스 규칙을 적용하여 운영 히스토리만 추적을 해보자는 방향으로 수정하였습니다.

### 최종 해결 방식
Dev 브랜치에는 개발자들이 수정된 내용이 있거나 테스트 할 내용을 merge(Fast-forward 방식) 을 합니다. 

테스트가 완료되면 Feature 브랜치를 Release 브랜치에 rebase를 하여  소스를 동기화 시켜줍니다.

배포 담당자는 Release 브랜치를 Prod 브랜치에 머지하여 운영 서버에 배포합니다.

이런 방식으로 전략을 유지하면 Prod 브랜치와 Release 브랜치는 깃 그래프를 봤을때 깔끔하게 보이는 것을 확인 할 수 있습니다.


- 명령어 정리
```bash
~ > git checkout [Feature Branch]
~ > git rebase [Release Branch]
~ > git commit 
~ > git checkout [Release Branch]
~ > git merge [Feature Branch]
~ > git push
```
