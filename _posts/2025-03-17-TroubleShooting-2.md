---
title: 20240423_TS
description: 20240423_TS
author: yoonxjoong
date: 2025-03-12 09:00:00 +0900
categories:
  - TroubleShooting
tags:
  - TroubleShooting
---
### 상황
- 로그인 페이지 진입 시에 기존 세션 정보가 남아 있음
- 특이한 케이스로 401 에러가 떨어져도 루트 페이지로 접속이 됨

### 원인
- 로그인 페이지 진입시에 기존 세션이 남아 있음 -> 세션을 초기화 해줄 필요가 있음
- 2차 인증 페이지로 접속 시 로그인이 된 사람은 접속이 가능함 -> 2차 인증 권한 유저만 페이지 접속이 가능해야될 필요가 있음
- 401 에러가 떨어지면 세션 만료 페이지로 이동하지 않음

### 조치
- 로그인 페이지 진입 시 세션을 초기화 해줄 필요가 있음
```java
@GetMapping("/login")  
public ModelAndView login(HttpServletRequest request) {  
    UserDetails userDetails = LoginHelper.getUserDetails();  

	// spring 컨텍스트 초기화
    if(!Objects.isNull(userDetails.getLoginId())){  
       SecurityContextHolder.clearContext();  
       // 로그아웃 핸들러을 이용하여 로그아웃 처리를 하려고 했지만 세션이 아예 없어서 401에러가 
       // 계속 떨어지는 이유로 컨텍스트만 초기화 하는거로 변경
       /*Authentication authentication = 
			       SecurityContextHolder.getContext().getAuthentication();  
		if (authentication != null) {  
		    // 로그아웃 핸들러를 사용하여 로그아웃 처리  
		    SecurityContextLogoutHandler logoutHandler = 
				    new SecurityContextLogoutHandler();  
		    logoutHandler.logout(request, response, authentication);  
		}*/
    }  
  
    ModelAndView mv = new ModelAndView();  
  
    return mv;  
}
```


-  2차 인증 권한 유저만 페이지 접속이 가능해야될 필요가 있음 -> 리다이렉션 처리
```java
public class CustomAuthenticationFilter extends OncePerRequestFilter {
	// 2차 인증 요청 권한이 아닌 사용자가 login2 페이지로 진입 시 리다이렉션을 해줌
	if(domain.equals("/login2") && !isAuthenticatedHalf(authentication)){  
    StringBuffer redirect = new StringBuffer();  
    if(useHttps) {  
        redirect.append("https://")  
                .append(request.getServerName());  
    }else {  
        redirect.append("http://")  
                .append(request.getServerName())  
                .append(':')  
                .append(request.getServerPort());  
    }  
    redirect.append("/login");  

    response.sendRedirect(redirect.toString());  
    return;  
}
```

- script에서 401 에러가 떨어지면 세션 만료 페이지로 이동 하도록 수정
```javascript
p.error = function(jqXHR, textStatus, errorThrown) {  
    if (jqXHR.status == 401) {  
       shop.isAuthorized = false;  
       if(p.url!="/login"){  
          $(location).attr('href', "/login?invalid=Y");  
       }else{  
          if(jqXHR.responseJSON){  
             if (!shop.text.allNull(jqXHR.responseJSON.message)) {  
                alert(jqXHR.responseJSON.message);  
             }  
          }  
          $(location).attr('href', "/login?invalid=Y");  
       }  
       return;  
    }
}
```