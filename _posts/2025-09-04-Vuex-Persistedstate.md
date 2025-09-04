---
title: "\bVuex의 저장한 데이터가 왜 초기화가 되지?"
description: "- 상황 - 경과(시도했던 것들) - 원인 - 조치 - 결과 순으로 작성"
author: yoonxjoong
date: 2025-09-04 09:00:00 +0900
categories:
  - TroubleShooting
tags:
---
안녕하세요. 현업에서 결제 모듈을 연동하는 업무를 담당하여 개발하고 있는 중에 트러블 슈팅에 대해 정리해보는 시간을 가져 보겠습니다.

저희 프로젝트는 Springboot + Vue 기반으로 되어있고, 결제 모듈은 PC 웹 / 모바일 웹 / 안드로이드 앱 / IOS 앱에서 연동되고 정상적으로 결제가 되어야 합니다.

결제 연동 과정에 대한 내용은 제가 이전에 정리한 포스트를 참고해주세요
[Spring boot와 Vue.js 환경에서 PG 결제 모듈 연동](https://yoonxjoong.github.io/posts/feature-paycoin-1/)


### 상황
안드로이드 앱에서 테스트를 하는 과정에 결제가 정상적으로 되지 않는 상황이 발생되었습니다.

PC 웹 / 모바일 웹/ IOS 앱까지 테스트를 마치고 이상이 없는 상황에서 안드로이드에서만 간헐적으로 결제가 되지 않더라구요.

로그를 확인해 보니, 결제 승인 후 PG 사에서 콜백 요청을 받아 처리하는 브릿지 페이지에서 Vuex에 저장된 결제를 위한 정보를 호출해서 확정 API 를 호출해야되는데 이게 안드로이드에서는 정보를 가져오지 못하는 거 같더라구요.
(앱 개발자와 테스트를 많이 해봤습니다..)

처음에는 안드로이드 앱에서만 결제가 실패하기 때문에 안드로이드 앱에서 데이터 처리를 제대로 못하고 있나? 라는 생각이 우선 들더라구요

## 원인

왜 그럴까? Vuex에 저장된 정보를 왜 가져오지 못하니?

안드로이드에서는 결제 모듈을 호출하고 복귀 할때 Store가 날라가는 현상에 있어 찾아보니 충분히 가능성이 있다는 점을 발견했습니다. 

그 이유로는 다음과 같이 정리했습니다.

**WebView 프로세스/메모리 관리 차이**

- 안드로이드 WebView는 외부 앱(예: PG 결제 모듈) 실행 후 복귀할 때, OS가 메모리를 회수하거나 WebView를 재생성할 수 있어요.
    
- 이 경우 Vuex는 메모리에만 저장되어 있기 때문에 초기화가 발생합니다.
    
- iOS는 상대적으로 WebView 세션 유지가 안정적이라 이런 현상이 덜해요.

**WebView 설정 문제 (`setSaveFormData`, `setDomStorageEnabled` 등)**

- Android WebView 기본 설정에서 LocalStorage/SessionStorage 동작이 제한될 수 있음.
    
- 특히 `domStorageEnabled`가 꺼져 있으면 `localStorage` 기반 persistence가 무용지물이 됩니다.

**Custom Tab / Intent 호출 시 세션 분리**

- 일부 PG 모듈이 `Custom Tab` 혹은 새로운 `WebView Activity`를 띄운 후 돌아오는데, 이 경우 원래 WebView와 다른 세션으로 복귀될 수 있음.
    
- 세션이 다르면 Vuex persistence layer에서 데이터 공유가 안 돼요.


### 조치
Vuex-persist를 LocalStorage가 아닌 indexedDB에 저장하는 방식으로 변경해보려고 합니다.

**indexed DB 특징**
- 브라우저/웹뷰 내부 **영속적 비동기 DB**
- 대용량 데이터 저장 가능, **세션/프로세스가 바뀌어도 삭제되지 않음**
- 안드로이드 WebView에서도 **WebView 재생성, Activity reload, Custom Tab 복귀 후에도 접근 가능**
- Vuex-persist를 indexedDB에 연결하면, 결제 모듈 호출 후 복귀해도 **데이터를 안전하게 읽어 올 수 있음**

## 결과

(작성 예정)