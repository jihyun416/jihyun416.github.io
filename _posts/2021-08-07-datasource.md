---
layout: post
title:  Springboot - Datasource (Master/Slave)
author: Jihyun
category: springboot
tags:
- springboot
- datasource
- hikari
- readonly
date: 2021-08-08 00:03 +0900
---

**Database Replication**은 중앙 Database를 하나 혹은 그 이상의 데이터베이스로 복제하는 것으로, **부하 분산 처리**를 위해 구성한다.

일반적으로 중앙 Database를 **Master**, 복제된 Database는 **Slave**로 지칭하며

복제는 Master에서 Slave로 **단방향**으로 이루어지기 때문에 Slave는 **읽기 전용**으로 사용한다.

데이터베이스의 작업 빈도는 쓰기 작업 보다 읽기 작업의 빈도가 높기 때문에, 

읽기 작업만 필요한 서비스는 Slave를 바라보게 하면 효과적인 분산 처리가 가능하다.



이 포스트에서는 Master Datasource와 Slave Datasource를 생성하고 서비스 트랜젝션이 읽기전용인지 여부에 따라 Datasource를 라우팅 하는 방법을 기술할 것이다.



## 1. 읽기, 쓰기 Enum 정의

```java
public enum DatabaseEnvironment {
    UPDATABLE,READONLY
}
```

- UPDATABLE : 쓰기 가능한 상태 (Master)
- READONLY : 읽기 전용(Slave)



## 2. 현재 원하는 상태를 저장할 DatabaseContextHolder 정의

```java
public class DatabaseContextHolder {
    private static final ThreadLocal<DatabaseEnvironment> CONTEXT = new ThreadLocal<>();

    public static void set(DatabaseEnvironment databaseEnvironment) {
        CONTEXT.set(databaseEnvironment);
    }

    public static DatabaseEnvironment getEnvironment() {
        return CONTEXT.get();
    }

    public static void reset() {
        CONTEXT.set(DatabaseEnvironment.UPDATABLE);
    }
}
```

- set: CONTEXT에 앞서 정의한 Enum을 이용하여 현재 원하는 상태를 세팅한다.
- getEnvironment: 현재 CONTEXT의 상태를 반환한다.
- reset: CONTEXT 상태를 UPDATABLE로 초기화한다. UPDATABLE은 읽기/쓰기가 가능하지만 READONLY는 읽기만 가능하기 때문에 더 포괄할 수 있는 UPDATABLE을 기본값으로 본다.

[](https://www.baeldung.com/hikaricp)

## 3. 라우팅을 담당 하는 MasterSlaveRoutingDataSource 정의

```java
public class MasterSlaveRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return DatabaseContextHolder.getEnvironment();
    }
}
```

- AbstractRoutingDataSource를 상속 받아 determineCurrentLookupKey를 재정의한다.
  - 앞서 정의한 DatabaseContextHolder에서 현재 원하는 상태를 가져와서 Datasource 선택 판단 근거로 한다.



## 4. Application.yml에 Master/Slave 연결 정보 정의

```yaml
datasource:
  master:
    jdbc-url: [your jdbc-url]
    username: [your username]
    password: [your password]
  slave:
    jdbc-url: [your jdbc-url]
    username: [your username]
    password: [your password]
```

- datasource.master 아래에 master 연결 정보를 입력
- datasource.slave 아래에 slave 연결 정보를 입력
  - datasource.master, datasource.slave 프로퍼티명은 변경되어도 무관하다. (해당 키를 기입하여 datasource에서 가져올 것이기 때문)
  - 하위 property값은 hikari에서 쓰는 명칭을 snake case로 맞춰서 쓰는 것이 좋다. (config를 이용하여 일괄 세팅하기 위해서는 명칭이 통일되어야 함, 이름이 다르면 setter를 이용하여 일일이 지정해야함)
  - 위 3가지 프로퍼티 외에도 다양한 config 값이 있으며 필요한 값을 추가하여 상세한 설정을 할 수 있다. [[HikariConfig]](https://www.baeldung.com/hikaricp)



## 5. DatabaseConfiguration 에서 DataSource 정의

```java
@Configuration
public class DataSourceConfiguration {
    @Bean
    @ConfigurationProperties(prefix = "datasource.master")
    public HikariConfig hikariMasterConfig() {
        HikariConfig config = new HikariConfig();
        return config;
    }

    public DataSource masterDataSource() {
        return new HikariDataSource(hikariMasterConfig());
    }

    @Bean
    @ConfigurationProperties(prefix = "datasource.slave")
    public HikariConfig hikariSlaveConfig() {
        HikariConfig config = new HikariConfig();
        return config;
    }

    public DataSource slaveDataSource() {
        return new HikariDataSource(hikariSlaveConfig());
    }
  
    @Bean
    public DataSource dataSource(){
        MasterSlaveRoutingDataSource masterSlaveRoutingDataSource = new MasterSlaveRoutingDataSource();
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put(DatabaseEnvironment.UPDATABLE, masterDataSource());
        targetDataSources.put(DatabaseEnvironment.READONLY, slaveDataSource());
        masterSlaveRoutingDataSource.setTargetDataSources(targetDataSources);
        masterSlaveRoutingDataSource.setDefaultTargetDataSource(masterDataSource());
        return masterSlaveRoutingDataSource;
    }
}
```

- ConfigurationProperties를 이용하여 HikariConfig에 yml에 설정한 프로퍼티들을 한번에 세팅한다. 

  - 이래서 이름을 맞춰주는 것이 중요!, 대응하는 이름이 헷갈리면 HikariConfig를 열어 변수를 확인한다.

- 만든 HikariConfig를 이용하여 HikariDatasource를 생성한다.

  - 하나씩 setter를 이용해서 설정하는 것도 가능하다.

    ```java
    @Value("${datasource.master.jdbc-url}")
    private String masterJdbcUrl;
    
    @Value("${datasource.master.username}")
    private String masterUsername;
    
    @Value("${datasource.master.password}")
    private String masterPassword;
    
    public DataSource masterDataSource() {
        HikariDataSource hikariDataSource = new HikariDataSource();
        hikariDataSource.setJdbcUrl(masterJdbcUrl);
        hikariDataSource.setUsername(masterUsername);
        hikariDataSource.setPassword(masterPassword);
        return hikariDataSource;
    }
    ```

- 앞서 정의한 MasterSlaveRoutingDatasource에 TargetDataSource를 세팅한다.
  - UPDATABLE에는 Master Datasource를, READONLY에는 Slave Datasource 정보를 세팅한다.
  - determineLookupKey에서 가져온 상태에 따라 Datasource를 제공하게 된다.
- 디폴트는 master로 세팅한다.



## 6. Context 상태 변경을 위한 Aspect 정의

```java
@Aspect
@Component
@Order(0)
public class TransactionReadonlyAspect {
    @Around("@annotation(transactional)")
    public Object proceed(ProceedingJoinPoint proceedingJoinPoint, Transactional transactional) throws Throwable {
        try {
            if (transactional.readOnly()) {
                DatabaseContextHolder.set(DatabaseEnvironment.READONLY);
            }
            return proceedingJoinPoint.proceed();
        } finally {
            DatabaseContextHolder.reset();
        }
    }
}
```

- Transaction 어노테이션을 감시하여 readonly로 설정된 Transaction일 경우 DatabaseContextHolder에 READONLY로 변경을 한다.
- 마지막에는 reset을 하여 UPDATABLE로 다시 돌려놓는다.
  - Transactional : import org.springframework.transaction.annotation.Transactional



## 7. Service Method에 Transaction Annotation 설정

```java
@RequiredArgsConstructor
@Service
public class UserService {
    public final UserRepository userRepository;
  
    @Transactional(readOnly = true)
    public User findUser(String userId) {
        return userRepository.findByUserId(userId).orElse(null);
    }
  
  ....
  
    @Transactional
    public User createUser(User user) {
        user.setUserSeq(null);
        return userRepository.save(user);
    }
}
```

- 읽기만 하는 서비스는 @Transactional(readOnly = true) 를 붙여준다.
- 쓰기를 포함하는 서비스는 @Transactional를 붙여준다.
  - Transactional : import org.springframework.transaction.annotation.Transactional



## 8. 테스트

- readOnly = true 여부에 따라 Datasource가 결정된다.
- 테스트를 위해 save가 있는 곳에 readonly로 설정하고 테스트를 해보면 읽기 전용 datasource를 타게 되어 오류를 만나게 된다.
  - Caused by: java.sql.SQLException: The MySQL server is running with the --read-only option so it cannot execute this statement



## 9. 주의할 점

- 요청을 기준으로 (Controller Method) 처음 Datasource가 필요한 순간 Datasouce를 가져온 뒤 요청이 끝날때 까지 같은 datasource를 사용한다. 

  따라서 컨트롤러 안에서 여러 서비스를 요청 할 경우 첫번째 서비스가 결정한 Datasource를 이용하기 때문에, 읽기/쓰기 작업이 섞여있는 경우 처음 쓰기 Datasource를 가져올 수 있게 주의가 필요하다.

**예시**

```java
@PostMapping("/test")
public UserDTO failTest(@RequestBody UserDTO userDTO) {
    userService.findUserList(userDTO);
    return this.convertToDto(userService.createUser(this.convertToEntity(userDTO)));
}
```

- 시나리오 : 컨트롤러 안에서 readonly로 설정한 findUserList를 호출한 뒤 readonly가 아닌 createUser를 호출한다.
- 원하는 결과
  - findUserList를 할때 READONLY로 컨텍스트를 바꾼 뒤 slave datasource를 가져오고, 완료 후 reset을 해서 UPDATABLE로 바꾼다.
  - createUser를 할 때 컨텍스트를 바꾸지 않은 뒤(readonly가 아니니까) *datasource를 가져오면 master datasource가 가져와져서 정상 실행된다.*
- 실제 결과
  - findUserList를 할때 READONLY로 컨텍스트를 바꾼 뒤 slave datasource를 가져오고, 완료 후 reset을 해서 UPDATABLE로 바꾼다.
  - createUser를 할 때 컨텍스트를 바꾸지 않는다.(readonly가 아니니까).
  - 다시 datasource를 가져오지 않기 때문에 컨텍스트를 바꾼게 무의미하며 readonly connection에서 save를 시도하니 오류가 난다.

- TransactionReadonlyAspect와 MasterSlaveRoutingDataSource breakpoint를 설정하고 디버그 시
  - findUserList 시작 전 TransactionReadonlyAspect.proceed try 타면서 Context 변경함
  - findUserList 안에 들어가서 실제 DB접근을 시작할 때 datasource를 가져오면서 결정하기 위해 MasterSlaveRoutingDataSource.determineCurrentLookupKey를 타는 것이 확인 됨
  - findUserList 종료 시 finally 타면서 reset 되는 것 확인됨
  - createUser 시작 전 TransactionReadonlyAspect.proceed try 탐
  - createUser 종료 시 finally 타면서 reset 됨



## 10. 추가

- 이 방법은 변경 기준을 Transaction을 어노테이션을 기준으로 AOP를 적용하였는데, AOP를 활용하여 패키지명, 메소드명을 이용하여 트랜젝션을 설정하고 컨택스트 변경을 할 수도 있다.
- 예를 들어 Service로 시작하는 클래스 중 find로 시작하는 것은 readonly를 적용한다<- 와 같이 정의하여 하나하나 어노테이션을 붙이지 않고 네이밍룰로 트랜젝션과 데이터소스 처리를 한번에 할 수 있다.
- 어떤 방법으로 하든 올바른 트랜젝션 및 데이터소스 처리를 위해 같이 개발하는 개발자들과의 약속이 필요하다.





#### 참고

> [Routing Read/Write Datasource in Spring. Thanh Tran](https://programmingsharing.com/routing-read-write-datasource-in-spring-99bcc4468f94)
