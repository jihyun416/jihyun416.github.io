---
layout: post
title:  Java - MapStruct
author: Jihyun
category: springboot
tags:
- springboot
- java
- mapstruct
- modelmapper
- dto
date: 2021-08-25 23:30 +0900
---

> MapStruct is a code generator that greatly simplifies the implementation of mappings between Java bean types based on a convention over configuration approach.



Java 개발을 하다보면 이러저러한 이슈로 서로 다른 클래스에 대해 내용 복제가 필요할 때가 있다. Entity <> DTO 간 전환이 대표적인 예라고 할 수 있다.

복제는 다양한 방법으로 할 수 있다.

1. Builder 혹은 Getter/Setter 노가다로 열심히 한땀한땀 복사하기
2. Reflection을 이용하여 setter함수 강제로 추측 후 invoke로 복제하기
3. 아주 똑똑한 사람들이 잘 만들어놓은 라이브러리 이용하기

생각나는 방법은 이정도가 있는데,

1번의 경우는 귀찮기도 하고 손품이 너무 많이 들며,

2번의 경우는 전에 구현을 해서 사용해본적이 있긴 한데.. java reflection은 퍼포먼스가 떨어진다고 한다. (*그리고 내가 구현한게 엄청 좋을리가ㅎㅎ*)

그래서 잘 만들어진것을 가져다 쓰려고 처음에 ModelMapper를 도입하려 했는데, 뭔가 Strict하게 일치하지 않는 클래스들을 매핑하려면 addMapping을 기술하는게 상당히 복잡했다.

그래서 다른 라이브러리를 알아보던 중 MapStruct를 발견했고, 가장 널리 쓰이고 있는 라이브러리라고 한다.

또한 ModelMapper는 Reflection을 이용하기 때문에 대량 작업 시에 성능이 좋지 않았다.

MapStruct는 인터페이스를 만들어주면 라이브러리가 annotation processor를 통해 구현체를 만들어주는데, 순수하게 setter를 이용하여 복사하게 때문에 속도가 빠르다.

두 라이브러리의 속도를 비교하기 위해 간단한 테스트를 진행하였다.



## ModelMapper vs MapStruct

```java
@SpringBootTest
public class ModelMapperAndMapStructTest {
    @Autowired
    TestMapper testMapper;
    @Autowired
    ModelMapper modelMapper;


    @DisplayName("ModelMapper 테스트")
    @Test
    public void ModelMapper() {
        TestDomain testDomain = TestDomain.builder().a("a").b("b").c("c").d("d").e("e").f("f").g("g").h(1).i(2).j(3).k(4).build();

        for(int i=0; i<1000000; i++) {
            TestDTO testDTO = modelMapper.map(testDomain, TestDTO.class);
        }
    }

    @DisplayName("MapStruct 테스트")
    @Test
    public void MapStruct() {
        TestDomain testDomain = TestDomain.builder().a("a").b("b").c("c").d("d").e("e").f("f").g("g").h(1).i(2).j(3).k(4).build();

        for(int i=0; i<1000000; i++) {
            TestDTO testDTO = testMapper.toDTO(testDomain);
        }
    }
}
```

### 결과

![](https://jihyun416.github.io/assets/springboot_9_1.png)

MapStruct의 압도적 승리다.



## build.gradle 의존성 추가

```yaml
dependencies {    
    implementation 'org.mapstruct:mapstruct:1.4.1.Final'
    compileOnly 'org.projectlombok:lombok'
    compileOnly 'org.projectlombok:lombok-mapstruct-binding:0.2.0'
    annotationProcessor 'org.projectlombok:lombok'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.4.1.Final'
}
```

- lombok 의존성도 함께 필요하다.



## 간단한 예제 (source와 target의 필드가 일치)

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface TestMapper {
    TestDTO toDTO(TestDomain test);
}
```

- **componentModel = "spring"** : spring에서 bean으로 사용하기 위해 추가해야 하는 속성이다. 쓰지 않으면 의존성 주입받아서 사용할 수가 없다.
  - Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException at DefaultListableBeanFactory.java:1790
- **unmappedTargetPolicy = ReportingPolicy.IGNORE** : target 필드 중 매칭되지 않는 필드가 있을 경우 무시하겠다는 이야기이다. 이 속성을 기입하지 않으면 매칭되지 않는 타겟 필드가 존재할 경우 warning을 만나게 된다.
  - warning: Unmapped target property



## 조금 복잡한 예제

```java
public class OrderDTO {
    String orderNumber;
    String customerFirstName;
    String customerLastName;
    String billingCity;
    String billingStreet;
    String dummyField;
}
```

```java
public class Order {
    String orderNumber;
    Customer customer;
    Address billing;
}
```

```java
public class Customer {
    Name name;
}
```

```java
public class Name {
    String firstName;
    String lastName;
}
```

```java
public class Address {
    String city;
    String street;
}
```

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {
    @Mapping(target = "dummyField", ignore = true)
    @Mapping(source = "customer.name.firstName", target="customerFirstName")
    @Mapping(source = "customer.name.lastName", target="customerLastName")
    @Mapping(source = "billing.city", target="billingCity")
    @Mapping(source = "billing.street", target="billingStreet")
    OrderDTO toDTO(Order order);

    @Mapping(target="customer.name", expression="java(Name.builder().firstName(orderDTO.getCustomerFirstName()).lastName(orderDTO.getCustomerLastName()).build())")
    @Mapping(target="billing", expression="java(Address.builder().city(orderDTO.getBillingCity()).street(orderDTO.getBillingStreet()).build())")
    Order toEntity(OrderDTO orderDTO);
}
```

- source는 파라미터로 받은 원래 객체이고 target은 결과물 객체이다.
- **target="필드명", ignore = true** : 개별 target에 대하여 매칭 작업을 무시하도록 처리할 수 있다.
- **source="필드명.필드명", target="필드명"** : 필드명이 다르거나 depth가 있는 경우 필드명을 기술하여 매칭시켜줄 수 있다.
- **target="필드명", expression="java(자바함수)"** : target에 들어갈 결과를 java함수를 이용하여 가공하여 넣을 수 있다.



## 주의할 점

- 기본 룰이 빡빡하다. 이름은 같지만 형태는 다른 경우 Mapping을 지정해주지 않으면 에러가 난다.
- 제대로 매칭되지 않을 경우 Annotation Processor 작업 중 에러가 날 수 있는데, 본인이 터져 놓고 남탓(?) 하는 척을 하기도 한다.
  - 타 프로젝트에서 Annotation Processor를 이용하는 Querydsl을 같이 사용중인데, Querydsl은 잘못이 없는데 QClass를 못찾는 척 에러메시지가 나왔었다. 알고보니 범인은 MapStruct... 만들때 신중을 기하고 빌드를 돌려보면서 구현체에 이상이 없는지 확인이 필요하다



뭔가 빡빡한 것 같으면서도 매핑 작업이 ModelMapper 보다 훨씬 직관적이고 명확하다.

Annotation processor가 돌고 나면 generated에서 구현체가 어떻게 생겼는지 구경 가능하다. expression 썼다가 에러메시지 빼애애액 하면 가서 확인해보면 된다.



#### 참고

> [MapStruct](https://mapstruct.org/)
>
> [Performance of Java Mapping Frameworks](https://www.baeldung.com/java-performance-mapping-frameworks)
>
> [천천히 올바르게, Spring Mapstruct - Java Entity DTO 매핑을 편하게 하자!](https://huisam.tistory.com/entry/mapStruct)
