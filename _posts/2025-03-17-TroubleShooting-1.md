---
title: 20240419_TS
description: 20240419_TS
author: yoonxjoong
date: 2025-03-12 09:00:00 +0900
categories:
  - TroubleShooting
tags:
  - TroubleShooting
---
## 상황
- 마스킹 처리를 위해서 어노테이션을 커스텀 하여 model에 선언하면 해당 변수만 선언된 타입에 따라서 마스킹 처리가 되는 로직을 개발함
- ObjectMapping에 마스킹하는 모듈을 등록하여 사용하기로 개발 방향을 잡음
- 기존에 설정된 ObjectMapping이 초기화 되는 현상이 발생

## 경과
```java
public class MaskingSample{
	@Masked(type = MaskingType.NAME)
	private String name;
	
	@Masked(type = MaskingType.CONTACT)
	private String telNo;

	@Masked(type = MaskingType.ADDRESS)
	private String dtlAddr;

	private String region;
}
```

위에 처럼 선언하면 name, tellNo, dtlAddr 의 값만 선언된 마스킹 규칙에 따라 마스킹 처리가 진행됨


- 커스텀 어노테이션 등록
```java
/**  
 * 마스킹 관련 커스텀 어노테이션 클래스  
 */  
@Target({ElementType.ANNOTATION_TYPE, ElementType.FIELD})  
@Retention(RetentionPolicy.RUNTIME) // 어노테이션 정보가 실행 시간까지 유지되도록 지정  
public @interface Masked {  
    MaskingType type();  
}
```

- 스프링 ObjectMapper에 마스킹 직렬화 클래스 모듈 등록
```java
@Configuration   
public class CustomObjectMapperConfig {  
    @Bean  
	public Jackson2ObjectMapperBuilderCustomizer customObjectMapper() {  
	    return builder -> {  
		    // 기존의 ObjectMapper 가져오기  
	        ObjectMapper objectMapper = builder.build(); 
	  
	        // 기존 ObjectMapper에 새로운 모듈을 추가  
	        SimpleModule module = new SimpleModule();  
	        module.addSerializer(String.class, new StringPropertyMasker());  
	        builder.modules(module);  
	    };  
	}
}
```

## 원인 
** 여기서 ObjectMapper에 커스텀 모듈을 등록하면 기존에 등록되어 있던 모듈이 초기화 되는 현상이 발생  **

```plain
위처럼 직접 빈 생성을 해주면 그 사이에 설정들을 포함할 수 있다. 이 방식을 사용할 경우 애플리케이션 초기 개발단계라면 큰 문제가 없을 수 있지만 이미 운영 중인 애플리케이션이라면 문제가 발생할 여지가 많다. 

spring boot 가 자동으로 생성하는 ObjectMapper 빈은 모든 설정이 초기화된 기본 인스턴스가 아니라 spring boot 에서 이미 다양한 설정들을 해놓은 인스턴스이기 때문이다. 

개발자가 직접 ObjectMapper 빈을 생성하는 순간 spring boot 는 더 이상 자동 구성되는 ObjectMapper 빈을 생성하지 않는데 그러면 애플리케이션 전역 ObjectMapper 빈이 직접 생성한 인스턴스로 변경되고, 자동 구성되는 설정들을 알게 모르게 이용하고 있었다면 이 설정들이 모두 바뀌게 되는 문제가 발생하기 때문이다. 

출처: https://multifrontgarden.tistory.com/300 [우리집앞마당:티스토리]
```

위에 내용을 참고하여 디버깅을 해본 결과 builder 에 있는 모듈이 초기화되고 새로 등록한 모듈만이 등록되어 있음을 확인을 함

테스트를 했을 당시 큰 오류는 없었지만 알지 못하는 곳에서 사용 될 수도 있다는 생각이 들어 다른 방식으로 구현하기로 결정했음.

## 조치
** reflect 클래스를 활용 ** 
reflect.field() 함수를 통해 등록된 어노테이션에 따라 마스킹 로직을 분기처리함

```java
public class Mask<T> {  
    private final MaskingMetaData maskingMetaData;  
  
    @Getter  
    private final T data;  
  
    public Mask(T data, Class<T> type) {  
        this.maskingMetaData = new MaskingMetaData(type);  
        this.data = data;  
        mastDataFields();  
  
    }  
  
    private void mastDataFields(){  
        for (String fieldName : maskingMetaData.getMaskingTypes()) {  
            MaskingType maskingType = maskingMetaData.getMaskingType(fieldName);  
            try {  
                Field field = data.getClass().getDeclaredField(fieldName);  
                field.setAccessible(true);  
                Object fieldValue = field.get(data);  
                if (fieldValue != null) {  
                    String maskedValue = MaskingUtil.maskingOf(maskingType, fieldValue.toString());  
                    field.set(data, maskedValue);  
                }  
            } catch (NoSuchFieldException | IllegalAccessException e) {  
                throw new RuntimeException("Error masking data field: " + e.getMessage());  
            }  
        }  
    }  
}


```

메타데이터에 어노테이션와 변수를 HashMap 형태로 저장을 하여 Mask 생성자 함수에서 호출하여 타입에 따라 마스킹 처리를 함

위에 방식대로 하면 @Congifure 클래스, ObjectMapper 를  변경할 필요가 없다는 장점이 있음

ObjectMapper 에서 등록된 모듈과 충돌 가능성이 줄어든다는 장점도 있음


마스킹 메서드 호출 방법
```java
Mask<MaskingInfo> mask = new Mask<>(obj, MaskingInfo.class);  
  
return mask.getData();
```