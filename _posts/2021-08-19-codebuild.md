---
layout: post
title:  AWS - CodeBuild를 이용하여 Springboot 프로젝트 빌드하기
author: Jihyun
category: aws
tags:
- aws
- codebuild
date: 2021-08-19 23:15 +0900
last_modified_at: 2021-08-20 14:36 +0900
---

> CloudWatch와 LambdaAWS CodeBuild는 소스 코드를 컴파일하고 테스트를 실행하며 배포 준비가 완료된 소프트웨어 패키지를 생성하는 완전 관리형 지속 통합 서비스입니다. CodeBuild를 사용하면 자체 빌드 서버를 프로비저닝, 관리 및 조정할 필요가 없습니다. CodeBuild는 지속적으로 조정되며 여러 빌드를 동시에 처리하기 때문에 빌드가 대기열에서 대기하지 않고 바로 처리됩니다.



AWS를 사용한다면 안쓸 이유가 없는 CI 도구인 Codebuild를 통해 Springboot 프로젝트를 빌드해보자.



## 1. buildspec.yml 작성

```yaml
version: 0.2 

phases:
  build: 
    commands:
      - echo Build stared on `date`
      - ./gradlew clean build
  post_build: 
    commands:
      - echo $(basename ./build/libs/*.jar) Build completed on `date`
      - echo $PROFILE >> profile.txt
      - pwd
artifacts:
  files:
    - build/libs/*.jar
    - profile.txt
    - appspec.yml
    - deploy.sh
  discard-paths: yes
cache:
  paths:
    - '/root/.gradle/caches/**/*'
```

- 프로젝트 루트에 build spec.yml 파일을 생성한다.

- phases.build.commands: 빌드 명령어를 넣는다.

  - 빌드 시작 로그를 찍기 위해 시간을 echo

  - ./gradlew clean build 명령어로 빌드를 실행하여 실행가능한 jar 파일을 만든다.

    > build 명령 시 plain.jar 가 생성되지 않도록 build.gradle에 하기 설정을 추가한다
    >
    > ```groovy
    > jar {
    >     enabled = false
    > }
    > ```

    - jar 파일명은 settings.gradle에 설정된 프로젝트명, build.gradle에 설정된 vesion 을 기준으로 생성된다

      프로젝트명-version.jar (e.g. userservice-1.0.0.jar)

- phases.post_build.commands : 빌드 후 실행될 명령어를 넣는다.

  - 빌드 완료 시간을 찍기 위해 echo
  - **추후 deploy시에 profile active할 때 쓸 profile정보를 환경변수로부터 받아 txt파일로 생성한다.**

- artifacts.files : 패키징할 산출물 파일들을 리스트로 넣는다.

  - build/libs/*.jar : 빌드된 실행가능한 jar 파일 (파일명은 변할수도 있으니까 *로 지정)
  - profile.txt : post_build 단계에서 만든 profile 정보를 담은 파일
  - appspec.yml, deploy.sh : deploy 단계에서 사용할 파일들 (deploy 포스팅 시 상세 설명)

- artifacts.discard-paths : yes -> 경로를 버리고 상위 루트에 파일을 다 넣겠다는 의미 *(안넣으면 jar 파일이 /build/libs 아래로 들어간다. 꺼내쓰기 귀찮으니 상위루트에 파일을 다 넣는 것으로 한다.)*

- cache.paths : 캐시 경로, s3 캐시 업로드 할 때 참조



## 2. S3 Bucket 생성

빌드 산출물을 저장할 S3 버킷을 생성한다. (자세한 설명 생략)



## 3. CodeBuild 프로젝트 생성

![](https://jihyun416.github.io/assets/aws_2_1.png)

- 프로젝트 이름을 지정한다

![](https://jihyun416.github.io/assets/aws_2_2.png)

- 소스를 가져올 공급자/리포지토리/브랜치를 지정한다.
- **CodePipeline을 통해 Codebuild 프로젝트를 생성할 경우, 소스를 이전 단계에서 지정하기 때문에 이 부분이 없다.**

![](https://jihyun416.github.io/assets/aws_2_3.png)

- 빌드할 환경을 설정한다.
- 기본적인 Amazon Linux 환경을 고른다

![](https://jihyun416.github.io/assets/aws_2_4.png)

- 프로젝트 생성 시 자동으로 역할이 생기도록 새 서비스 역할을 지정한다.

![](https://jihyun416.github.io/assets/aws_2_5.png)

- 추가 구성에서 buildspec.yml 에서 받아서 쓸 환경변수를 지정한다.
  - 이 codebuild project로 빌드한 결과물은 dev로 active 하기 위해 PROFILE=dev 라는 환경변수를 넣었다. *(이름 상관없음. 추후 Deploy를 위해 한 방법일뿐 필수 아님)*
  - **CodePipeline을 이용하여 Codebuild를 실행할 경우 CodePipeline에서도 환경변수 지정이 가능하다.**

![](https://jihyun416.github.io/assets/aws_2_6.png)

- 프로젝트 내의 buildspec.yml을 사용하겠다고 설정

![](https://jihyun416.github.io/assets/aws_2_7.png)

- 결과물을 올릴 S3 와 이름을 지정한다.
- Zip으로 패키징해서 업로드한다.

![](https://jihyun416.github.io/assets/aws_2_8.png)

- CloudWatch 로그를 선택한다. (선택해야 빌드할때 나오는 로그 확인 가능함)



#### 생성한 프로젝트에 들어가서 [빌드 시작] 을 해서 정상 작동하는지 확인하면 끝!







### 참고

> [두발로걷는개의 발자국, AWS CodeBuild로 빌드 후 S3에 빌드 결과파일 업로드](https://twofootdog.tistory.com/37)

