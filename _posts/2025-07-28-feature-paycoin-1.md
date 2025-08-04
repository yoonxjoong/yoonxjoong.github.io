---
title: Spring boot와 Vue.js 환경에서 PG 결제 모듈 연동
description: 
author: yoonxjoong
date: 2025-08-04 09:00:00 +0900
categories:
  - ARCHITECTURE
tags:
---
운영중인 프로젝트에서 결제 모듈을 연동하는 업무를 맡았습니다.  기존 백엔드에서 브릿지 페이지를 활용한 구조에서  프론트에서(Vue.js) 브릿지 페이지를 통해 인증값을 주고 받는 구조로 변경한 개발기를 기록한 문서입니다.

### PG 모듈 연동 : 브릿지 페이지를 이용한 결제 모듈 연동

- 결제 흐름도

![[Pasted image 20250804163749.png]]

1. 사용자가 결제 요청 버튼을 클릭합니다.
2. 버튼을 클릭하면 결제 승인 API 호출하여 결제 프로세스를 시작합니다.
3. 결제 승인 API 응답값이 성공일 경우 인증 성공 콜백 함수 호출, 실패일 경우 인증 실패 콜백 함수를 호출합니다. 
4. 인증 콜백 함수는 URL 값으로 성공 실패 값을 포함시켜 브릿지 페이지로 리다이렉션을 합니다.
5. 브릿지 페이지에서는 유효성 검증을 하여 결제 확정 API 를 호출하여 결제 완료를 합니다.

위와 같은 다이어그램을 직관적으로 결제 흐름을 정확하게 파악할수 있겠죠?
PG를 제공하는 여러 업체가 있겠지만, 위와 같은 흐름을 파악하면 다른 업체들도 쉽게 연동을 할 수 있을거에요.

우선 첫번째 단계부터 차근차근 살펴봅시다

#### 결제 승인 API 호출

결제 프로세스에 있어 가장 첫 번째 관문입니다. 각 PG사에 결제를 승인받는 API 에요. 가맹점을 인증하고, 사용자 인증시 사용되는 정보를 전달하는 API 에요. 
PG 사에서 제공하는 키값, 결제 금액, 타입 등 결제에 필요한 정보들을 전달합니다. 

스니핑 공격에 방어하기 위해 전문은 각 PG 사에서 제공하는 암호화 방식을 사용하여 암호화 합니다. 
이렇게 되면 악의적인 사용자가 데이터 전문을 확인 할 수 없겠죠?

정상적인 요청에 의해 승인이 떨어지면, 결제 창을 띄울수 있는 정보로 응답을 받습니다. 

위에 부분이 결제 승인 프로세스 입니다. 

정리하면 결제를 하기 위한 정보를 PG 사에 승인을 받는 프로세스로 생각하면 될 거 같아요.


#### PG 결제창을 호출

결제 승인이 성공한다면? 응답값으로 결제창을 호출할 수 있는 정보를 보내줄거에요. 이를 활용해서 form데이터를 호출해 결제창을 띄우면 됩니다. 

저의 개발 환경은 Vue.js이기 때문에  Vue 개발 환경으로 설명을 드릴게요

``` Vue.js
// 결제창 호출을 위한 Form 데이터
<form  
  ref="orderForm"  
  :action="orderResponse.starturl.toString()"  
  method="post"  
  style="display: none;"  
>  
  <input  
    type="hidden"  
    name="tid"  
    :value="orderResponse.tid.toString()"  
  >  
  <input
	type="hidden"  
	name="back_url"  
	:value="encodedBackUrl"  
  >  
  <input
    type="hidden"  
    name="startparam"  
    :value="encodedStartParam"  
  >  
</form>


// orderResponse 변화가 감지되면 orderForm을 제출하여 결제창을 호출 합니다.
watch(orderResponse, (newValue) => {  
  if (newValue) {  
    // DOM 업데이트를 기다린 후 실행  
    nextTick(() => {  
      if (orderForm.value) {  
        orderForm.value.submit()  
      } else {  
        console.error('리다이렉트 폼 요소를 찾을 수 없습니다.')  
      }  
    })  
  }  
})
```

#### 결제창 안에서 결제 후 콜백 메서드 호출

2단계 이후 결제를 한다면 결제창은 정상 콜백 API (/callback/success) 를 호출하게 됩니다. 
``` java
/**  
 * 사용자 인증 성공 시 콜백 함수 * @param request  
 * @param redirectParam  
 * @return  
 * @throws NoSuchAlgorithmException  
 * @throws InvalidKeyException  
 */@PostMapping(value = "/callback/success", consumes = "application/x-www-form-urlencoded")  
public RedirectView handlePaycoinCallbackSuccess(  
        @ModelAttribute OrderPayCoinCallBackRequest request,  
        @RequestParam(value = "redirect", required = false) String redirectParam)  
        throws NoSuchAlgorithmException, InvalidKeyException {  
  
    return orderPayCoinService.processCallBackSuccess(redirectParam, request.getTid());  
}
```


반대로 실패할 경우 다음과 같은 실패 콜백(/callback/fail) API를 호출합니다.
``` java
/**  
 * 사용자 인증 실패 시 콜백 함수 * @param request  
 * @param redirectParam  
 * @return  
 * @throws NoSuchAlgorithmException  
 * @throws InvalidKeyException  
 */@PostMapping(value = "/callback/fail", consumes = "application/x-www-form-urlencoded")  
public RedirectView handleCallbackFail(  
        @ModelAttribute OrderCallBackRequest request,  
        @RequestParam(value = "redirect", required = false) String redirectParam)  
        throws NoSuchAlgorithmException, InvalidKeyException {  
  
    return orderService.processCallBackFail(redirectParam);  
}
```

여기서 특이점은 `리턴 타입` 이 RedirectView라는 점인데요, 

>RedirectView 는 스프링 프레임 워크에서 제공하는 View 구현체로, 클라이언트의 요청을 다른 URL에 리다이렉트하는 데 사용됩니다.

결제 승인이 성공하면 콜백 성공 API를 호출하여 브릿지 페이지에 성공 값을 URL파라미터에 포함시켜 리다이렉트를 할 것이고, 실패하면 실패 값을 URL 파라미터에 포함시켜 리다이렉트를 할 것입니다.

#### 브릿지 페이지에서 결제 확정 API 호출

RedirectView에서 호출된 Bridge 화면에서는 인증작업을 통해 정상적인 요청에 대하여 결제 확정 API를 호출합니다.

결제에 있어 최종 단계로 확정을 짓는 단계 입니다.  각 PG에 요구하는 데이터를 전송하고 내부 프로세스에서는 결제 완료에 대한 처리를 하면 되죠.

예를 들어 상태값 변경, 알림 전송, 이력 저장 등, 내부 프로세스에서 사용되는 로직을 추가함으로써 최종 결제는 완료되는 것이지요.

```Vue.js
// 브릿지 페이지가 마운트 될 때 성공 / 실패 여부를 구분하여 데이터 처리
onMounted(async () => {  
  try {  
    const { status } = route.query  
  
    // 결제 취소 콜백  
    if (status === 'fail') {  
      await validationCallBackFail()  
    } else if (status === 'success') {  
      await validationCallBackSuccess()  
    } else {  
      setTimeout(() => {  
        router.replace('/')  
      }, 3000)  
    }  
  } catch (err) {  
    console.error('Payment Bridge Error', err)  
    setTimeout(() => {  
      router.replace('/')  
    }, 3000)  
  } finally {  
	// 데이터 초기화
  }})
```

#### 정리

위에 글을 통해 간단하게 결제 모듈을 연동하고 어떻게 결제가 되는지 흐름을 파악하셨을 거 같아요. 결제 모듈을 담당하여 개발하다 보면 이러한 프로세스를 파악하여 설계하는것도 중요하지만, 보안적인 측면에서 좀 더 신경을 써서 개발을 해야될거에요.

이번 시간에는 결제 흐름에 대해서 정리하는 포스팅이기 때문에 보안적인 내용은 빼고 작성해봤어요.

다음 글에 어떻게 보안적으로 안전하게 결제를 진행할수 있을까에 대한 내용에 대해 같이 공유해보도록 하겠습니다.

