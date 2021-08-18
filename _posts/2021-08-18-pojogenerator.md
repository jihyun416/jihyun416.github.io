---
layout: post
title:  Intellij - Generate POJOs.groovy
author: Jihyun
category: IntelliJ
tags:
- IntelliJ
- jpa
- pojo
- entity
- groovy
date: 2021-08-18 21:43 +0900
---

JPA로 구현을 하기 위해서는 **DB Table Schema**와 대응되도록 **Entity**를 생성해야 한다.

개발언어에서 사용하는 타입과 DBMS에서 사용하는 타입에는 차이가 있으며,

컬럼명도 일반적으로 Entity에서는 Camel Case를, DB에서는 Snake Case를 사용하는데, 이를 변환하는 일은 꽤나 번거로운 작업이다. (카멜케이스 변환기 이런 도구가 있긴 하지만...)



JPA Option을 통해 자동으로 DDL이 생성되도록 하면, Entity에 대응되는 DB 테이블을 자동으로 생성하는 것도 가능하지만, 

원치 않는 타입으로 생성된다던지 (String의 경우 VARCHAR(n)으로 하고 싶을 수도 있고 TEXT로 하고 싶을 수도 있다)

자동으로 생성된 Constraint의 이름이 맘에 안든다던지,

순서가 맘에 안든다던지 등의 이유로 개별적으로 생성하는 경우가 많다.



특히나 DB 중심의 모델링에 익숙한 경우 DB ERD를 먼저 그리는 경우가 많은데,

이에 맞춰 엔티티를 수기로 만들었고, 어쩔수 없이 노가다 해야 하는 부분이라고 생각했는데



IntelliJ... 역시 인텔리하다... *(또 멍청한건 나뿐이었지...🤯)*

IntelliJ 안에 내장된 Database (DataGrip) 을 통해 POJO 객체를 생성 가능하다.



> *JPA와 IntelliJ를 2년넘게 써오면서 몰랐던 기능인데... 이제 막 JPA와 IntelliJ 사용을 시작한 Anthony님의 제보로 알게되었다...* 
>
> *아마 Anthony님이 궁금해 하지 않았다면 난 평생 이런 기능을 모르고 살았을지도 모른다... 없다고 생각했겠지!*
>
> *타성에 젖는다는게 이런거구나 꺼이꺼이 급 반성*😭



## 1. 생성 스크립트 확인

## 

![finder](https://jihyun416.github.io/assets/intellij_1_1.png)

![script](https://jihyun416.github.io/assets/intellij_1_2.png)

- Database 마우스 오른쪽 -> Scripted Extensions -> Go to Scripts Directory (Project에서 해당 경로가 열림)

  ->Generate POJOs.groovy 파일 열기



## 2. 기본적으로 설정되어 있는 Generate POJOs.groovy

```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil

/*
 * Available context bindings:
 *   SELECTION   Iterable<DasObject>
 *   PROJECT     project
 *   FILES       files helper
 */

packageName = "com.sample;"
typeMapping = [
  (~/(?i)int/)                      : "long",
  (~/(?i)float|double|decimal|real/): "double",
  (~/(?i)datetime|timestamp/)       : "java.sql.Timestamp",
  (~/(?i)date/)                     : "java.sql.Date",
  (~/(?i)time/)                     : "java.sql.Time",
  (~/(?i)/)                         : "String"
]

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
  SELECTION.filter { it instanceof DasTable }.each { generate(it, dir) }
}

def generate(table, dir) {
  def className = javaName(table.getName(), true)
  def fields = calcFields(table)
  new File(dir, className + ".java").withPrintWriter { out -> generate(out, className, fields) }
}

def generate(out, className, fields) {
  out.println "package $packageName"
  out.println ""
  out.println ""
  out.println "public class $className {"
  out.println ""
  fields.each() {
    if (it.annos != "") out.println "  ${it.annos}"
    out.println "  private ${it.type} ${it.name};"
  }
  out.println ""
  fields.each() {
    out.println ""
    out.println "  public ${it.type} get${it.name.capitalize()}() {"
    out.println "    return ${it.name};"
    out.println "  }"
    out.println ""
    out.println "  public void set${it.name.capitalize()}(${it.type} ${it.name}) {"
    out.println "    this.${it.name} = ${it.name};"
    out.println "  }"
    out.println ""
  }
  out.println "}"
}

def calcFields(table) {
  DasUtil.getColumns(table).reduce([]) { fields, col ->
    def spec = Case.LOWER.apply(col.getDataType().getSpecification())
    def typeStr = typeMapping.find { p, t -> p.matcher(spec).find() }.value
    fields += [[
                 name : javaName(col.getName(), false),
                 type : typeStr,
                 annos: ""]]
  }
}

def javaName(str, capitalize) {
  def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
    .collect { Case.LOWER.apply(it).capitalize() }
    .join("")
    .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
  capitalize || s.length() == 1? s : Case.LOWER.apply(s[0]) + s[1..-1]
}
```

- **packageName** : class 상단에 package 선언 시 넣을 패키지 명을 넣는다.
- **typeMapping** : DB 에서 사용하는 타입명 - Java에서 사용하는 타입명을 매칭시킨다.
- **FILES.chooseDirectoryAndSav**e : 저장 위치를 받고 선택한 테이블들을 대상으로 엔티티 저장 함수를 호출한다.
  - Generate POJOs.groovy를 실행시키면 생성된 파일을 저장할 위치를 선택하라는 탐색기가 나온다.
  - src 폴더 아래 도메인을 관리하는 패키지를 선택한다.
  - 선택한 파일들을 순회하면서 generate를 호출한다. (한꺼번에 여러 테이블 엔티티 생성 가능!)
- **def generate(table, dir)** : table과 저장경로를 파라미터로 받아 실질적으로 파일을 생성한다.
  - 테이블에서 테이블명을 추출하여 자바스타일로 이름을 바꾼다.
  - calcFields 를 호출하여 컬럼 처리를 한다.
  - generate(out, className, fields) 를 호출하여 파일 내용을 구성하고, 자바스타일로 이름 바꾼거.java로 파일을 쓴다.
- **def generate(out, className, fields)** : 파일 내용을 구성한다.
  - out : outputStream을 호출자로부터 받는다.
  - className : 클래스명 부분에서 사용
  - fields : 컬럼별로 타입, 이름을 담은 리스트이다.
  - out.println 을 이용하여 위에서 받은 정보들을 토대로 가공하여 클래스 내용을 구성한다.
- **def calcFields(table)** : 테이블을 변수로 받아 테이블 내의 컬럼을 순회하며, 컬럼명을 자바에서 쓰는 Camel Case로 변환하고, typeMapping에서 선언한 내용을 토대로 자바 타입을 찾아준다.
- **def javaName(str, capitalize)** : str을 Camel Case로 변환해준다. capitalize가 true이면 첫글자를 대문자로, false면 소문자로 반환한다. (클래스명 일 땐 대문자, 멤버필드명 일 땐 소문자 사용)



#### 파일 생성

![save](https://jihyun416.github.io/assets/intellij_1_3.png)

Database 마우스 오른쪽 -> Scripted Extensions -> Generate POJOs.groovy

클릭 시 파일 위치 지정하라는 창 뜨면 Entity 생성하는 src 하위 폴더 선택하고 Open

(위 groovy 스크립트를 실행시키는 것이라고 생각하면 된다!)



```java
package com.sample;


public class User {

  private String userId;
  private java.sql.Timestamp editDatetime;
  private String editId;
  
  ...

  public String getUserId() {
    return userId;
  }

  public void setUserId(String userId) {
    this.userId = userId;
  }
  
  public java.sql.Timestamp getEditDatetime() {
    return editDatetime;
  }

  public void setEditDatetime(java.sql.Timestamp editDatetime) {
    this.editDatetime = editDatetime;
  }
  ....
}
```

- 기본 템플릿으로 생성 시 나온 Output이다.
- 패키지명도 다르고, 사용하는 타입도 다르고 Lombok을 사용할 것이기 때문에 Getter/Setter는 필요없다!
- 커스텀 해보자!



## 3. 입맛대로 바꿔본 Generate POJOs.groovy

```groovy
import com.intellij.database.model.DasTable
import com.intellij.database.util.Case
import com.intellij.database.util.DasUtil

/*
* Available context bindings:
*   SELECTION   Iterable<DasObject>
*   PROJECT     project
*   FILES       files helper
*/

packageName = "com.jessy.user.domain;"
typeMapping = [
        (~/(?i)bigint/)                   : "Long",
        (~/(?i)tinyint/)                  : "Boolean",
        (~/(?i)int/)                      : "Integer",
        (~/(?i)float|double|decimal|real/): "Double",
        (~/(?i)datetime|timestamp|time/)  : "LocalDateTime",
        (~/(?i)time/)                     : "LocalTime",
        (~/(?i)date/)                     : "LocalDate",
        (~/(?i)/)                         : "String"
]

FILES.chooseDirectoryAndSave("Choose directory", "Choose where to store generated files") { dir ->
    SELECTION.filter { it instanceof DasTable }.each { generate(it, dir) }
}

def generate(table, dir) {
    def tableName = table.getName()
    def className = javaName(tableName, true)
    def fields = calcFields(table)
    new File(dir, className + ".java").withPrintWriter { out -> generate(out, tableName, className, fields) }
}

def generate(out, tableName, className, fields) {
    out.println "package $packageName"
    out.println ""
    out.println "import lombok.*; "
    out.println "import javax.persistence.*;"
    out.println ""
    out.println "@Builder"
    out.println "@Getter"
    out.println "@Setter"
    out.println "@NoArgsConstructor"
    out.println "@AllArgsConstructor"
    out.println "@Entity"
    out.println "@Table "
    out.println "public class $className" + " extends BaseEntity {"
    fields.each() {
        if (it.name == className.uncapitalize() + "Seq") {
            out.println "   @Id"
            out.println "   @GeneratedValue(strategy = GenerationType.AUTO)"
        }
        if (it.name == className.uncapitalize() + "Id") {
            out.println "   @Id"
        }
        if (!(it.name == "useFlag"
                || it.name == "registerId"
                || it.name == "registerDatetime"
                || it.name == "updateId"
                || it.name == "updateDatetime")
        ) {
            out.println "   private ${it.type} ${it.name};"
        }
    }
    out.println "}"
}

def calcFields(table) {
    DasUtil.getColumns(table).reduce([]) { fields, col ->
        def spec = Case.LOWER.apply(col.getDataType().getSpecification())
        def typeStr = typeMapping.find { p, t -> p.matcher(spec).find() }.value
        fields += [[
                           name   : javaName(col.getName(), false),
                           oriName: col.getName(),
                           type   : typeStr,
                           annos  : ""]]
    }
}

def javaName(str, capitalize) {
    def s = com.intellij.psi.codeStyle.NameUtil.splitNameIntoWords(str)
            .collect { Case.LOWER.apply(it).capitalize() }
            .join("")
            .replaceAll(/[^\p{javaJavaIdentifierPart}[_]]/, "_")
    capitalize || s.length() == 1 ? s : Case.LOWER.apply(s[0]) + s[1..-1]
}
```

- packageName : 내 패키지명으로 변경
- typeMapping : 타입 변환
  - boxing 타입으로 변환 (JPA에서 boolean, int 이런거 쓰면 지옥문 오픈)
  - 일시와 관련된 부분은 LocalDatetime, LocalDate, LocalTime 으로 매칭 (java.time.*)
- generate(out, tableName, className, fields) 내용 변경
  - import 추가 (Local 타입들은 쓰일수도 있고 안쓰일수도 있어서 import java.time.* 는 일단 제외했다. 있으면 생성 후 추가 import 하면 되니까!)
  - Annotation 추가 (Lombok, Entity, Table)
  - class 라인에서 extends BaseEntity 부분 추가
  - 필드 순회 시 클래스명+Seq 이거나 클래스명+Id 일 경우 pk 일 가능성이 유력하므로 @Id Annotation 추가되도록 설정
  - BaseEntity에 있는 필드들이 생성되지 않도록 BaseEntity에 있는 필드가 아닐경우에만 out되도록 처리



#### 위와 동일한 방법으로 파일 생성

```java
package com.jessy.user.domain;

import lombok.*; 
import javax.persistence.*;

@Builder
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table 
public class User extends BaseEntity {
   @Id
   private String userId;
   private String password;
   private String userName;
   private String email;
   private String phoneNumber;
   private Integer attemptCount;
   private LocalDateTime lastLoginDatetime;
   private String lastPassword;
   private LocalDateTime passwordChangeDatetime;
   ...
}
```

- 대강 원하는 모양새가 뽑혔다!



#### 아쉬운점

- 관계 매핑까지는 어려울듯... DasUtil.getColumns에서 얻을수 있는 정보가 타입과 이름밖에 없는듯함*(아...아닐수도)*



#### 참고

> 자동으로 다 해줬으면 하는 Anthony님의 제보

