---
layout: post
title:  Springboot - RestControllerAdvice로 Exception 처리하기
author: Jihyun
category: springboot
tags:
- springboot
- exception
- advice
date: 2021-08-01 00:20 +0900

---



예외처리에 대해 별도 설정하지 않을 경우 아래와 같이 기본적으로 처리된다.



#### 1) 브라우저를 통해 호출한 경우 : 

text/html 타입으로 Whitelabel Error Page를 응답받음

![Whitelabel Error Page](https://jihyun416.github.io/assets/springboot_4_1.png)



#### 2. Postman을 통해 호출한 경우

application/json 타입으로 에러 json을 응답받음

![json type response](https://jihyun416.github.io/assets/springboot_4_8.png)



이와 같은 예외처리를

- 내가 정한 일정한 포맷으로

- 익셉션 종류에 따라서 걸맞는 HTTP STATUS 헤더와 함께

- 컨트롤러 메소드별로 하나씩 설정하지 않고
- 항상 json 포맷으로

응답하고자 한다.



#### @ControllerAdvice

> Specialization of @Component for classes that declare @ExceptionHandler, @InitBinder, or @ModelAttribute methods to be shared across multiple @Controller classes.



#### @RestControllerAdvice

>A convenience annotation that is itself annotated with @ControllerAdvice and @ResponseBody.
>Types that carry this annotation are treated as controller advice where @ExceptionHandler methods assume @ResponseBody semantics by default.



#### @ExceptionHandler

> Annotation for handling exceptions in specific handler classes and/or handler methods.



#### @ResponseStatus

> Marks a method or exception class with the status code and reason that should be returned.



이 목적을 달성하기 위해

**@RestControllerAdvice + @ExceptionHandler + @ResponseStatus** 의 조합으로

**ExceptionAdvice**를 생성하여 처리한다.



## 1. 응답 패턴 ResponseDTO 생성

예외 상황이 발생하였을 때 응답할 객체를 정의

```groovy
@Getter
@Setter
@AllArgsConstructor
@Builder
@JsonInclude(JsonInclude.Include.NON_EMPTY)
public class ResponseDTO {
    private Boolean result; // 성공여부
    private Integer status; // HTTP 상태
    private String message; // 메시지
    private Object data; // 데이터
}
```

- Exception 발생 시, 일관적으로 이 DTO에 내용을 담아 전달한다.
- Exception 외에도, 단순 성공 응답이나, 에러는 없었으나 실패인 상황에서 특정한 메시지를 전달하기 위해 같은 객체를 활용할 수 있다.



## 2. ExceptionAdvice 추가

```java
@Slf4j
@RestControllerAdvice
public class ExceptionAdvice {
    @ExceptionHandler(value={AccessDeniedException.class, AuthenticationEntryPointException.class})
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ResponseDTO AuthenticationEntryPointException(AccessDeniedException e) {
        ResponseDTO responseDTO = ResponseDTO.builder()
                .status(HttpStatus.FORBIDDEN.value())
                .result(false)
                .message(e.getClass().getName())
                .data(e.getMessage())
                .build();
        return responseDTO;
    }

    @ExceptionHandler(NoHandlerFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ResponseDTO noHandlerFoundException(NoHandlerFoundException e) {
        log.error(e.getMessage());
        ResponseDTO responseDTO = ResponseDTO.builder()
                .status(HttpStatus.NOT_FOUND.value())
                .result(false)
                .message(e.getClass().getName())
                .data(e.getMessage())
                .build();
        return responseDTO;
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ResponseDTO handleException(Exception e) {
        StringWriter sw = new StringWriter();
        PrintWriter pw = new PrintWriter(sw);
        e.printStackTrace(pw);
        String sStackTrace = sw.toString();
        log.error(sStackTrace);
        ResponseDTO responseDTO = ResponseDTO.builder()
                .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
                .result(false)
                .message(e.getClass().getName())
                .data(sStackTrace)
                .build();
        return responseDTO;
    }
}
```

- 클래스에 @RestControllerAdvice를 붙인다. (특정 클래스나 패키지 단위로 제한도 가능, 안쓰면 전체 적용)
- @ExceptionHandler에 매칭시킬 Exception class를 선언한다. 여러 클래스를 같은 메소드로 처리할 수 있다.
- @ResponseStatus에 응답 헤더 HTTP STATUS에 넣을 값을 선언해준다.
- 위에서 생성한 ResponseDTO를 활용하여 응답한다.




## 3. 예외 발생 테스트

#### 1) 예외를 강제로 발생시키는 Controller 작성

```java
@RequestMapping("/exception")
@RestController
public class ExceptionTestController {
    @GetMapping("/runtime")
    public String exceptionTest500() {
        throw new RuntimeException();
    }

    @GetMapping("/noauth")
    public String exceptionTest403() throws AccessDeniedException {
        throw new AccessDeniedException("Access Denied.");
    }
}
```

+ 404 NoHandlerFoundException은 없는 아무 패스를 호출하여 테스트한다.



#### 2) 테스트 결과

![403](https://jihyun416.github.io/assets/springboot_4_3.png)

![500](https://jihyun416.github.io/assets/springboot_4_4.png)

- HTTP STATUS와 함께 지정한 양식으로 json 응답이 온다. (브라우저에서 테스트 해도 application/json 타입으로 응답됨)



그런데 말입니다...

**404**의 경우는 **ExceptionAdvise**를 타지 않고 이전과 동일한 결과를 리턴.....



## 4. 404(NoHandlerException) 처리를 위한 추가 처리

#### 1) @EnableWebMvc 선언

```
@EnableWebMvc
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {
}
```

- 완전한 제어를 위해 EnableWebMvc 추가
- WebMvcConfigurer와 같이 있을 필요는 없지만 추후 Swagger 설정 시 Mvc 관련 설정을 추가 할 것이기 때문에 이곳에 같이 설정해둠



#### 2) application.yml에 설정 추가

```yaml
spring:
  mvc:
    throw-exception-if-no-handler-found: true
```

- handler가 없을 경우 exception을 던지도록 해야 advice에 감지된다.



#### 3) 테스트 결과

![404](https://jihyun416.github.io/assets/springboot_4_7.png)

- ExceptionAdvice를 정상적으로 타는 모습 확인



#### 참고

> [Incheol's tech blog @EnableWebMvc](https://incheol-jung.gitbook.io/docs/q-and-a/spring/enablewebmvc)
