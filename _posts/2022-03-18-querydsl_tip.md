---
layout: post
title:  QueryDsl Tip #1
author: Jihyun
category: springboot
tags:
- springboot
- querydsl
date: 2022-03-18 18:14 +0900
---
누군가에겐 당연히 알고 있고 사용하고 있는 방법일 수 있지만 회사에서 의미를 모른채 하던 습관을 개선하기 위해 작성한 글입니다.


## 1. QEntity를 import static으로 가져오기

#### **👀Before**

```java
import com.bcm.lesson.entity.QLesson;
import com.bcm.lesson.entity.QLessonFeedback;

public class LessonFeedbackRepositoryImpl {
    private QLesson lesson = QLesson.lesson;
    private QLessonFeedback lessonFeedback = QLessonFeedback.lessonFeedback;
    ...
}
```

#### **✨After**

```java
import static com.bcm.lesson.entity.QLesson.lesson;
import static com.bcm.lesson.entity.QLessonFeedback.lessonFeedback;

public class LessonFeedbackRepositoryImpl {
	...
}
```

QueryDsl로 자동 생성된 QEntity를 따라가 보면

```java
public static final QLesson lesson = new QLesson("lesson");
```

 위와 같이 static으로 QEntity가 생성되었음을 알 수 있습니다.

변수를 재선언할 필요 없이 import static을 이용하면 깔끔하게 처리 가능합니다.

다만 이 변수는 엔티티명과 동일하게 alias를 주어 생성된 객체이므로, 다른 alias의 변수가 필요한 경우 내부에서 new로 생성합니다.

```java
public class TeacherWorktimeRepository {
    private QTeacherWorktime teacherWorktime1 = new QTeacherWorktime("worktime1");
    private QTeacherWorktime teacherWorktime2 = new QTeacherWorktime("worktime2");
    private QTeacherWorktime teacherWorktime3 = new QTeacherWorktime("worktime3");
    ...
}
```



## 2. extends QuerydslRepositorySupport는 필수가 아니다

#### **👀Before**

```java
public class LessonFeedbackRepositoryImpl extends QuerydslRepositorySupport implements LessonFeedbackRepositoryCustom {
    private final JPAQueryFactory queryFactory;

    public LessonFeedbackRepositoryImpl(JPAQueryFactory queryFactory) {
        super(LessonFeedback.class);
        this.queryFactory = queryFactory;
    }
    ...
}
```

#### **✨After**

```java
@RequiredArgsConstructor
public class LessonFeedbackRepositoryImpl implements LessonFeedbackRepositoryCustom {
    private final JPAQueryFactory queryFactory;
    ...
}
```

QuerydslRepositorySupport는 스프링 데이터가 제공하는 페이징을 Querydsl로 변환할 때 사용하는 클래스 입니다.

이를 사용하기 위해서 상속받고 페이징 대상이 되는 클래스를 상위 생성자 호출에 넣어주고, queryFactory가 의존성 주입받을 수 있게 선언을 해주었는데, Page를 사용할 일이 없다면 굳이 상속받을 이유가 없습니다.

따라서 구현체에  Page가 없는경우는 extends QuerydslRepositorySupport를 지우고, JPAQueryFactory를 주입하기 위해 간단하게 @RequiredArgsConstructor로 대체합니다.

간혹 Page를 사용하는 구현체도 있습니다! 그런 부분은 상속받아야 합니다.



## 3. Repository, RepositoryCustom, RepositoryImpl 구조에 대한 고찰

![img](https://jihyun416.github.io/assets/springboot_3_2.png)

#### DomainRepository

```java
public interface LevelFeedbackRepository extends JpaRepository<LevelFeedback, Long>, LevelFeedbackRepositoryCustom {
    Optional<LevelFeedback> findTopByFreelessonSeqEquals(Long freelessonSeq);
}
```

#### DomainRepositoryCustom

```java
public interface LevelFeedbackRepositoryCustom {
    Optional<LevelReference> findCurrentLevel(Long memberSeq);
}
```

#### DomainRepositoryImpl

```java
...
@RequiredArgsConstructor
public class LevelFeedbackRepositoryImpl implements LevelFeedbackRepositoryCustom {
    private final JPAQueryFactory queryFactory;

    @Override
    public Optional<LevelReference> findCurrentLevel(Long memberSeq) {
        ...
    }
}
```

현재는 QueryDSL이 필요한 모든 Repository의 구성을 위와 같이 3개의 파일로 구성하고 있습니다.

JPA Repository가 interface인 특성을 이용하여, JPA Repository를 주입받았을때 QueryDSL로 선언한 메소드도 같이 활용하기 위해 Custom interface를 만들어 repository에서 extends하고 Custom의 구현체를 Impl로 만드는 형식입니다.

이 방법을 이용한다면 DomainRepository에 주입받은 bean은 DomainRepository의 행동과 QueryDSL로 선언한 행동 모두를 함께 할 수 있습니다.

```java
private final LevelFeedbackRepository levelFeedbackRepository;
// Repository에 선언한 메소드 호출 가능
levelFeedbackRepository.findTopByFreelessonSeqEquals(lessonSeq);
// Impl에서 구현한 QueryDSL 메소드 호출 가능
levelFeedbackRepository.findCurrentLevel(memberSeq);
```



하지만 어떤 경우에는 특정 JPA repository에 종속시키기 어렵거나, 굳이 JPA repository가 필요하지 않은 경우도 있습니다.

그럴 경우에는 QueryDSL 구현체만 독립적으로 선언하여 사용할 수 있습니다.

```java
@RequiredArgsConstructor
@Repository
public class CalendarRepository {
    private final JPAQueryFactory queryFactory;

    public List<Map<String, Object>> findWeekCalendar(Integer year) {
       ...
    }
}
```

- Custom 인터페이스를 굳이 만들 필요가 없습니다. 따라서 implements를 생략하였습니다.
- @Repository를 선언해 주어야 합니다.

