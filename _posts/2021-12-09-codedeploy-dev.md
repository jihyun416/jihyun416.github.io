---
layout: post
title:  AWS - CodePipeline(CodeCommit, CodeDeploy)를 이용한 EC2 단순 배포 구성
author: Jihyun
category: aws
tags:
- aws
- codecommit
- codedeploy
- codepipeline
- ec2
- docker
- docker-compose
date: 2021-12-09 16:40 +0900
---

개발환경에서는 도커이미지를 따로 관리하지 않고 소스코드를 받아서 바로 도커컴포즈를 이용하여 프로세스를 가동해 볼 것이다. (빌드해서 이미지 저장소로 푸시하는 과정이 없음)

따라서 소스 저장소에서 소스를 가져오고(SourceArtifact) 해당 소스를 서버로 가져와(Install) 쉘스립트를 통해 어플리케이션을 실행하는 간단한 구조로 CI/CD를 구성해 보았다.



![](https://jihyun416.github.io/assets/aws_12_3.png)



## EC2 IAM Role 설정

![](https://jihyun416.github.io/assets/aws_12_1.png)

- 필요한 정책을 연결한 역할(Role)을 생성한다.
  - SecretsManagerReadWrite : 어플리케이션에서 ORM 설정 정보를  Secrets Manager에서 가져오기 위해 필요
  - AmazonS3FullAccess : SourceArtifact를 가져오고 환경변수 파일을 다운로드 받기 위해 필요
  - CloudWatchFullAccess : Deploy 시 CloudWatch에 로그를 쌓기 위해 필요
  - AWSCodeDeployFullAccess : CodeDeploy 사용을 위해 필요

![](https://jihyun416.github.io/assets/aws_12_2.png)

- 배포 대상의 EC2에 생성한 역할을 연결한다.
- 이전에 연결한 역할이 있다면 해당 역할에 정책을 추가하거나 전 역할의 정책도 같이 포함하는 역할을 생성 후 교체해준다.



## EC2 서버 환경 세팅

```shell
sudo yum update -y
sudo amazon-linux-extras install docker
sudo service docker start
sudo usermod -a -G docker ec2-user
sudo chkconfig docker on
sudo curl -L https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
sudo yum install ruby
sudo yum install wget
cd /home/ec2-user
wget https://aws-codedeploy-ap-northeast-2.s3.ap-northeast-2.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto
sudo service codedeploy-agent status
```

- 도커와 도커컴포즈 설치
- CodeDeploy Agent 설치



## CodeDeploy 어플리케이션/배포그룹 생성

### 1) AWSCodeDeployRole 생성

![](https://jihyun416.github.io/assets/aws_12_7.png)

- AWSCodeDeployRole 정책을 포함하는 역할이 없을 경우 생성한다.

### 2) CodeDeploy 어플리케이션 생성

![](https://jihyun416.github.io/assets/aws_12_8.png)

- 컴퓨팅 플랫폼을 **EC2/온프레미스**를 선택하여 어플리케이션을 생성한다.

### 3) 배포그룹 생성

![CodeDeploy Flow](https://docs.aws.amazon.com/ko_kr/codedeploy/latest/userguide/images/lifecycle-event-order-in-place.png)

- 좌측에 로드밸런서 없이 배포하는 로직대로 수행할 것이다.
  - 개발서버에도 로드밸런서가 있긴 하지만, 로드밸런서의 대상 그룹에 EC2가 1개밖에 없기 때문에 우측 방식이 무의미하다.

![](https://jihyun416.github.io/assets/aws_12_9.png)

- 배포 그룹의 이름을 입력한다.
- 위에서 생성한 AWSCodeDeployRole을 서비스 역할로 연결한다.
- 배포 유형은 현재위치를 선택한다. (현재 EC2에 바로 배포함)

![](https://jihyun416.github.io/assets/aws_12_10.png)

- Amazon EC2 인스턴스를 선택한다. (Auto Scaling 그룹이 아니고 단일 EC2에 할것이기 때문)
- EC2 인스턴스에 붙인 태그를 기준으로 대상 인스턴스를 추가한다. (Name태그 기준으로 추가해줌)

![](https://jihyun416.github.io/assets/aws_12_11.png)

- 로드밸런서는 사용하지 않기때문에 체크하지 않는다. (우측 방식을 이용할 경우 체크)



## CodeDeploy 관련 파일 생성(프로젝트 내에 추가)

### 1) appspec.yml

```yaml
version: 0.0
os: linux
files:
  - source: ./
    destination: /소스를설치할경로
    overwrite: yes
hooks:
  BeforeInstall:
    - location: ./beforeInstall.sh
      timeout: 900
      runas: root
  ApplicationStart:
    - location: ./deploy.sh
      timeout: 900
      runas: root
```

- 배포 구성 파일이다.

- Source Artifact로 받은 소스를 **/소스를설치할경로**에 설치한다.

- BeforeInstall은 설치 이전 시 실행되는 단계이다.

  #### 2) beforeInstall.sh

  ```shell
  rm -rf /home/ec2-user/bio-backend
  mkdir /home/ec2-user/bio-backend
  ```

  - 기존 소스 폴더를 삭제 후 폴더를 다시 생성한다. (기존 소스 삭제 역할)

- ApplicationStart는 애플리케이션 실행 단계이다.

  #### 3) deploy.sh

  ```shell
  CURRENT_PS=$(docker ps -a -q)
  if [ -z "$CURRENT_PS" ]
  then
    echo There is no docker container.
  else
    docker rm -f $CURRENT_PS
  fi
  CURRENT_IMG=$(docker images -a -q)
  if [ -z "$CURRENT_PS" ]
  then
    echo There is no docker image.
  else
    docker rmi -f $(docker images -a -q)
  fi
  
  cd /home/ec2-user/bio-backend
  aws s3 cp s3://application-env/bio-dev.env /home/ec2-user/bio-backend
  mv bio-dev.env .env
  docker-compose -f deployments/docker-compose.dev.yaml up -d --build
  ```

  - 기존 도커 컨테이너를 삭제한다.
  - 기존 도커 이미지를 삭제한다.
  - S3에서 환경변수 파일을 받아서 프로젝트에 넣는다.
  - 도커컴포즈를 이용해 실행한다.



## CodePipeline 구성

### 1) Source

![](https://jihyun416.github.io/assets/aws_12_4.png)

- 대상 소스의 리포지토리와 브랜치를 선택한다.
- 저장소의 소스가 SourceArtifact로 S3에 업로드 된다.

### 2) 빌드 단계 건너뛰기

### 3) Deploy

![](https://jihyun416.github.io/assets/aws_12_6.png)

- 작업공급자로 AWS CodeDeploy를 선택한다.
- 입력 아티팩트는 SourceArtifact를 선택한다. (빌드를 건너뛰었기 때문에 빌드는 선택할수도 없음!)
  - Install 단계에서 SourceArtifact를 다운로드 받는다.

### 4) 편집으로 Source와 Deploy 사이에 수동 승인 추가

![](https://jihyun416.github.io/assets/aws_12_5.png)

- 수동 승인 단계를 추가한다. (모든 develop push에 대해 배포가 되지 않게 하기 위함)
- 수동 승인은 처리하지 않으면 7일 뒤에 자동으로 실패 처리된다.
- 수동 승인을 처리하지 않고 또 소스 변경이 발생한다면 병목현상이 발생한다. 배포하지 않으려는 건수는 승인거절 처리를 해야 하며, CD가 설정되어있는 브랜치에는 배포를 하지 않을 것이라면 가급적 푸시를 하지 않는 것이 좋다. (feature 브랜치 이용 후 배포 필요 시 merge)