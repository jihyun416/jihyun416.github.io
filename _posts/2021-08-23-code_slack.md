---
layout: post
title:  AWS - Code Commit/Build/Deploy/Pipeline Slack 알림 설정
author: Jihyun
category: aws
tags:
- aws
- codepipeline
- codecommit
- codebuild
- codedeploy
- slack
date: 2021-08-23 09:05 +0900
---



CodeCommit, CodeBuild, CodeDeploy, CodePipeline에서는 알림 규칙 생성을 통해 리소스에서 발생하는 이벤트에 대해 구독 설정을 할 수 있다. 

구독은 SNS 또는 AWS Chatbot으로 가능하며, AWS Chatbot을 이용하여 Slack과 알림 연동을 해 볼 것이다.

![](https://jihyun416.github.io/assets/aws_5_2.png)

![](https://jihyun416.github.io/assets/aws_5_1.png)

- 알림 이름 설정
- 세부 정보 유형은 가득참(Full) 선택
- 대상은 AWS Chatbot 선택하여 대상으로 등록해 놓은 Slack 채널 선택



### CodeCommit Event

![](https://jihyun416.github.io/assets/aws_5_4.png)



### CodeBuild Event

![](https://jihyun416.github.io/assets/aws_5_5.png)



### CodeDeploy Event

![](https://jihyun416.github.io/assets/aws_5_6.png)



### CodePipeline Event

![](https://jihyun416.github.io/assets/aws_5_7.png)



### 대상 설정

![](https://jihyun416.github.io/assets/aws_5_8.png)

- 알림 대상이 미리 정의되어 있지 않은 경우 [대상 생성]을 이용하여 Slack을 추가할 수 있다.

![](https://jihyun416.github.io/assets/aws_5_9.png)

![](https://jihyun416.github.io/assets/aws_5_10.png)

![](https://jihyun416.github.io/assets/aws_5_11.png)

![](https://jihyun416.github.io/assets/aws_5_12.png)

![](https://jihyun416.github.io/assets/aws_5_13.png)

![](https://jihyun416.github.io/assets/aws_5_14.png)



### 알림 예시



#### CodeCommit

![](https://jihyun416.github.io/assets/aws_5_15.png)



#### CodeBuild

![](https://jihyun416.github.io/assets/aws_5_16.png)



#### CodeDeploy

![](https://jihyun416.github.io/assets/aws_5_17.png)



#### CodePipeline

![](https://jihyun416.github.io/assets/aws_5_18.png)



알림을 트리거하는 이벤트를 모두 선택할 경우, 파이프라인 한 번 가동 시 알림이 너무 많이 울리게 되므로 취사 선택하여 사용한다.

나의 경우는 CodeCommit은 알림 규칙보다는 Lambda를 이용하여 구현한 것이 더 자세한 정보를 노출할 수 있어서 [CodeCommit 이벤트는 Lambda를 이용한 알림을](https://jihyun416.github.io/aws/2021/08/18/codecommit_trigger/) 활용하고, 

CodeBuild에서는 Build State를, 

CodeDeploy에서는 Deployment를,

CodePipeline에서는 Pipeline Execution을 이용하여

파이프라인은 시작과 끝을 알리도록 설정하고, 각 단계에는 각 서비스가 알림을 하게 최종 설정 하였다.

