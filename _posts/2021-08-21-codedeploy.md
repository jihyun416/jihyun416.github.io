---
layout: post
title:  AWS - CodeDeploy를 이용하여 Springboot 프로젝트 배포하기
author: Jihyun
category: aws
tags:
- aws
- codedeploy
date: 2021-08-21 22:20 +0900
---

> AWS CodeDeploy는 소프트웨어 배포를 자동화하는 완전 관리형 배포 서비스로써 Amazon EC2, AWS Lambda 및 온프레미스 서버를 컴퓨팅합니다. AWS CodeDeploy를 사용하면 새로운 기능을 더욱 쉽고 빠르게 출시할 수 있고, 애플리케이션을 배포하는 동안 가동 중지 시간을 줄이는 데 도움이 되며, 복잡한 애플리케이션 업데이트 작업을 처리할 수 있습니다.



CodeDeploy는 배포 자동화를 도와주는 서비스로 EC2, Lambda, ECS에 배포를 할 수 있다.

배포유형은 In-place(현재위치), Blue/Green 방식을 지원한다.

In-place 방식은 기존의 환경에 새로운 어플리케이션을 배포하는 것을 의미하며

Blue/Green 방식은 새로운 환경에 새로운 어플리케이션을 배포한 뒤 트래픽을 교체해주는 방식이다.



![In-place lifecycle event order](https://docs.aws.amazon.com/codedeploy/latest/userguide/images/lifecycle-event-order-in-place.png)

> The **Start**, **DownloadBundle**, **Install**, and **End** events in the deployment cannot be scripted, which is why they appear in gray in this diagram. However, you can edit the `'files'` section of the AppSpec file to specify what's installed during the **Install** event.

In-place 방식 중 로드밸런서가 존재하지 않는 경우에는 단순 배포가 되며, 로드밸런서가 있는 경우에는 트래픽 차단 및 허용 과정이 추가된다. 로드밸런서를 이용하는 경우 로드밸런서 이하의 인스턴스들을 Rolling 방식으로 번갈아가면서 트래픽차단-배포-트래픽허용 하기 때문에 중단 없이 배포가 가능하다.



배포에 사용할 appspec.yml 파일에서는 회색을 제외한 부분에서 수행할 명령어에 대해 정의할 수 있다.



## 1. appspec.yml

```yaml
version: 0.0 
os: linux 
files: 
  - source: ./ 
    destination: /deploy/myproject
    overwrite: yes

hooks:
  ApplicationStart:
    - location: ./deploy.sh
      timeout: 60
      runas: root
 
```

- install 단계에서 배포할 파일에 대한 정의를 files에 정의한다
  - **source** : codedeploy에서 입력 받은 revision(개정), 파일 혹은 디렉토리 지정 가능, 디렉토리를 지정하여 아티팩트 내 모든 파일을 옮긴다.
  - **destination** : 아티팩트 파일을 올릴 위치
  - **overwrite: yes** : destination에 배포 시 파일이 있을 경우 overwrite 하겠다는 의미, 배포를 한 뒤에 destination에 파일이 남아있을 경우 다음 배포시에 install 단계에서 파일이 이미 있다는 오류와 함께 배포가 되지 않기  때문에 덮어 쓸 수 있도록 지정해준다.
- hooks 중 ApplicationStart 단계에서 할 일을 정의한다.
  - **location: ./deploy.sh** : 해당 단계에서 실행할 스크립트의 위치. deploy.sh 내용은 아래 설명
  - **timeout** : 스크립트가 실패했다고 판단할 기준 초수, 설정하지 않으면 기본값은 3600초(1시간)이다. 정상 처리된다면 오래걸릴 일이 없는 스크립트이므로 짧게 60초로 변경해준다.
  - **runas** : 스크립트 실행 시 사용하는 사용자, root로 실행되도록 한다.



## 2. deploy.sh

```shell
#!/bin/bash

cd /deploy/myproject
ProFile=`cat profile.txt`
pid=`ps -ef | grep myservice | grep -v grep | awk '{print $2}'`
kill -9 $pid
sleep 1
cp /deploy/myproject/myservice-0.0.1.jar /deploy/
nohup java -jar -Xms256m -Xmx512m -Dspring.profiles.active=$ProFile /deploy/myservice-0.0.1.jar &> /dev/null &
```

- destination 위치로 이동한다.
- build 단계에서 실행시킬 profile을 profile.txt에 저장하도록 지정했었다. 그 파일을 읽어서 Profile 변수에 받는다.
- 기존 서비스를 실행한 프로세스를 찾아 강제 종료한다.
- 실행시킬 jar 파일을 실행시킬 위치로 복사한다.
- 백그라운드로 java를 실행시킨다.



위 두 파일은 프로젝트 루트에 저장하여, codebuild시에 아티팩트로 같이 패키징 되도록 처리한다.





## 3. 배포할 EC2의 IAM 역할 수정

![ec2](https://jihyun416.github.io/assets/aws_3_1.png)

![ec2 role](https://jihyun416.github.io/assets/aws_3_2.png)

- CodeDeploy 대상이 될 인스턴스에서 우클릭  [보안] - [IAM 역할 수정] 을 통해 IAM 정책을 수정한다.
  - 기존에 지정된 역할이 있을 경우 기존 역할에 필요한 역할을 추가한다.
  - 기존에 지정된 역할이 없을 경우 새로운 역할을 생성하여 연결시킨다.
  - 배포파일을 가져오기 위한 S3 접근 권한과, CodeDeploy와의 연동을 위한 CodeDeploy 권한, 로그 처리를 위한 CloudWatch 권한이 필요하다.



## 4. codedeploy 에서 사용할 IAM 역할 생성

![codedeployrole](https://jihyun416.github.io/assets/aws_3_3.png)

- CodeDeploy 배포 그룹 생성 시 사용할 서비스역할을 생성한다.
- AWSCodeDeployRole이 필요하다.



## 5. CodeDeploy Agent 설치

[아마존 리눅스 또는 RHEL용 CodeDeploy 에이전트 설치](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/codedeploy-agent-operations-install-linux.html)

위 문서를 참고하여 EC2에 CodeDeploy Agent를 설치한다.



## 6. CodeDeploy 애플리케이션 생성

![code deploy application](https://jihyun416.github.io/assets/aws_3_4.png)

- 애플리케이션 이름을 입력한다.
- 컴퓨팅 플랫폼에서 배포 환경을 선택한다. (EC2, Lambda, ECS 중 선택)
- 애플리케이션 생성 단계에서 하는 일은 이게 다다! 실질적으로 애플리케이션 아래 배포그룹에서 타켓을 지정해준다.
- 애플리케이션 하위에는 여러 배포그룹을 지정할 수 있다. 따라서 동일한 어플리케이션인데 Stage만 다른 경우 (develop, stage, production), 같은 애플리케이션 그룹 아래 배포그룹으로 관리하는것이 용이하다.

![codedeploy application](https://jihyun416.github.io/assets/aws_3_5.png)

- 생성하면 지정한것이라고는 이름과 컴퓨팅 플랫폼뿐인 어플리케이션이 생성된다. [배포 그룹 생성]을 통해 실질적으로 배포가 일어날 배포그룹에 대해 지정한다.



## 7. 배포 그룹 생성

![deploy group](https://jihyun416.github.io/assets/aws_3_6.png)

- **배포 그룹 이름** : 배포 그룹 명을 지정한다. 같은 애플리케이션 내에서 고유한 이름을 가지면 되므로 그룹을 특정할 수 있는 이름을 지어주면 좋다.
- **서비스 역할** : 위에서 codedeploy 에 대한 정책을 준 역할을 입력해준다.
- **배포 유형** : 현재 위치 (In-place) 선택, 블루/그린의 경우는 컴퓨팅 자원이 추가로 들기 때문에 있는 것에 덮어 쓰는 옵션으로 진행



![deploy group](https://jihyun416.github.io/assets/aws_3_7.png)

- 환경 구성을 선택해준다. 현재 환경에 Auto Scaling이 지정되어 있지 않으므로 EC2 인스턴스만 지정해줬다.  Auto Scaling 되어 있는 환경이라면 Auto Scaling 그룹도 추가로 지정해준다.
- 키 아래 인스턴스 태그를 추가하여 어떤 인스턴스인지 매칭한다.



![deploy group](https://jihyun416.github.io/assets/aws_3_8.png)

- CodeDeploy 에이전트에 대해 업데이트 설정을 한다. (에이전트 설치 자체는 5번에서 했듯이 따로 해줘야 함)
- 배포 설정은 한번에 배포하는 CodeDeployDefault.AllAtOnce 를 지정한다. (사실 1개의 인스턴스라 의미 없는거 같..) [자세한 설명은 여기 참고](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/deployment-configurations.html)

- 로드 밸런서 : 현재 로드밸런서가 없기 때문에 로드 밸런싱 활성화는 체크를 해지하였다. 로드밸런서가 있을 경우는 체크를 한다. 로드밸런스가 있으면 트래픽 차단, 트래픽 허용 부분이 추가된다.



배포 생성을 이용하여 개정(Revision) 파일을 수기 지정하여 배포를 실행해 볼 수 있다.



다음 단계에서는 CodePipeline으로 CodeCommit-CodeBuild-CodeDeploy를 연결하여 푸시 발생 시 자동 배포가 되도록 할 것이다.



#### 참고

> [AWS CodeDeploy User Guide](https://docs.aws.amazon.com/codedeploy/latest/userguide/reference-appspec-file-structure-hooks.html#reference-appspec-file-structure-hooks-list)
