---
layout: post
title:  AWS - CodePipeline를 이용하여 CI/CD 구축하기
author: Jihyun
category: aws
tags:
- aws
- codepipeline
- codecommit
- codebuild
- code deploy
date: 2021-08-22 22:33 +0900
---

> AWS CodePipeline은 빠르고 안정적인 애플리케이션 및 인프라 업데이트를 위한 지속적 통합 및 지속적 전달 서비스입니다. CodePipeline은 사용자가 정의한 릴리스 프로세스 모델에 따라 코드가 변경될 때마다 코드를 빌드, 테스트 및 배포합니다.



## CodePipeline에서 선택할 수 있는 작업 공급자

#### 소스

- AWS CodeCommit
- Amazon ECR
- Amazon S3
- Bitbucket
- GitHub Enterprise Server
- GitHub(Version 1)
- GitHub(Version 2)

#### 빌드

- AWS CodeBuild
- Jenkins 추가

#### 배포

- AWS AppConfig
- AWS CloudFormation
- AWS CloudFormation 스택 세트
- AWS CloudFormation 스택 인스턴스
- AWS CodeDeploy
- AWS Elastic Beanstalk
- AWS OpsWorks 스택
- AWS Service Catalog
- Amazon ECS
- Amazon ECS(Blue/Green)
- Amazon S3

#### 승인

- 수동 승인

#### 테스트

- AWS CodeBuild
- AWS Device Farm
- BlazeMeter
- Ghost Inspector UI 테스트
- Jenkins 추가
- Micro Forcus StormRunner 로드
- Runscope API 모티터링

#### 호출

- AWS Lambda
- AWS Step Functions



CodePipeline에서는 다양한 작업 공급자를 이용하여 파이프라인을 설정할 수 있다. 이 예제에서는 CodeCommit, CodeBuild, CodeDeploy를 이용하여 파이프라인을 생성해 볼 것이다.



![파이프라인 생성](https://jihyun416.github.io/assets/aws_4_1.png)

- [파이프라인 생성] 을 통해 새로운 파이프라인 만들기를 시작한다.



![파이프라인 설정 선택](https://jihyun416.github.io/assets/aws_4_2.png)

- 파이프라인 이름을 설정한다.
- 서비스 역할을 새로 정의하거나 기존 서비스 역할을 불러온다.



![소스 스테이지 추가](https://jihyun416.github.io/assets/aws_4_3.png)

- CodeCommit의 특정 리포지토리, 브랜치를 소스 공급자로 선택한다.
- CodeCommit 변경 감지를 권장 값으로 설정한다. 변경이 감지되면 파이프라인이 작동된다.



- CodeBuild 프로젝트를 CodePipeline을 통해 들어가서 생성하면 소스 공급자를 제외하고 설정할 수 있다.

![빌드 프로젝트 생성](https://jihyun416.github.io/assets/aws_4_4.png)

![](https://jihyun416.github.io/assets/aws_4_5.png)

![](https://jihyun416.github.io/assets/aws_4_6.png)



![](https://jihyun416.github.io/assets/aws_4_7.png)

- 빌드 스테이지에서 생성한 CodeBuild 프로젝트를 추가한다.
- CodeBuild의 환경변수를 파이프라인에서 추가할 수 있다. CodeBuild 프로젝트는 다른 파이프라인에서도 재사용하고, 환경변수는 파이프라인마다 달리 주기 위해 이곳에서 환경변수를 주는 것으로 설정했다.



![](https://jihyun416.github.io/assets/aws_4_8.png)

- 배포 스테이지에서는 기존에 생성한 CodeDeploy 프로젝트를 이용한다.
- 어플리케이션 이름과 배포그룹을 선택한다.



![](https://jihyun416.github.io/assets/aws_4_9.png)

- Step 별로 선택한 설정을 검토하고 파이프라인을 생성한다.



![](https://jihyun416.github.io/assets/aws_4_10.png)

- 파이프라인이 생성되었다.
- 파이프라인이 가동될 때 작동하는 모습을 살펴볼 수 있다.





#### 참고

> [AWS Documentation CodePipeline](https://docs.aws.amazon.com/ko_kr/codepipeline/latest/userguide/welcome.html)
