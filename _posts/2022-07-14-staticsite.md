---
layout: post
title: AWS - 정적 웹사이트 호스팅
author: Jihyun
category: aws
tags:
- aws
- s3
- cloudfront
- static site
- react
date: 2022-07-14 15:10 +0900
---
SPA react 프로젝트는 서버가 필요없는 정적 웹사이트이기 때문에 다양한 배포 옵션을 선택할 수 있다.

그중 AWS를 이용할 경우 S3 + CloudFront를 이용하면 저렴하고 간단하게 웹호스팅이 가능하다.

덤으로 CodeCommit과 Codebuild를 이용하여 푸시할 경우 자동으로 빌드+배포가 되도록 함께 구성해보았다.



## 1. S3 버킷 생성

- 빌드 산출물(html, js)을 올릴 S3 버킷을 생성한다.
- 추후 새로운 개정 배포 시 기존 개정으로 롤백을 용이하게 하기 위한 백업 버킷도 추가로 생성한다.
- S3 버킷 생성 과정은 간단하므로 절차는 생략한다.
- 퍼블릭 엑세스를 허용하지 않고 Default 값으로 생성한다.



## 2. CloudFront 설정

### 1) 배포생성

![](https://jihyun416.github.io/assets/aws_13_01.png)

- CloudFront에 접속하여 [배포생성]을 클릭

![](https://jihyun416.github.io/assets/aws_13_02.png)

- 원본 도메인에 생성한 S3 버킷을 선택한다.
- 이름은 원본 도메인을 선택하면 자동으로 지정된다.
- S3 버킷 엑세스를 [예, OAI사용(버킷은 CloudFront에 대한 엑세스로만 제한 가능)]을 선택한다.
  - S3에 퍼블릭 엑세스 없이 CloudFront을 통해서만 접근 가능!
  - 새 OAI 생성을 클릭하여 새로운 아이디를 생성
  - [예, 버킷 정책 업데이트]를 선택하여 S3 버킷의 정책 업데이트 (*직접은 힘들다....정책 어려웡...*)

![](https://jihyun416.github.io/assets/aws_13_03.png)

- 자동으로 객체 압축은 No 를 선택 (단순히 이미지 같은것만 서빙하는게 아니기 때문에 혹시나 코드에 문제가 생길까봐!)
- 뷰어 프로토콜 정책은 Redirect HTTP to HTTPS 를 선택하여 http로 접속 시 https로 리다이렉트 되도록 처리한다.
- 허용된 HTTP 방법은 사실 GET, HEAD만 있어도 잘 실행될 것으로 보이나(단순히 페이지 다운로드 하는 것이므로) 혹시 몰라 모든 메소드를 허용!
- 캐시 키 및 원본 요청은 추천 방식을 선택
  - 잦은 배포를 감안하여 캐싱이 되지 않게 하는 방법도 있으나 Cloudfront의 이점을 살리지 못함! (CloudFront는 CDN 서비스)
  - 캐시를 사용하되 배포할 경우 캐시 무효화를 이용하여 기존 캐싱된 파일을 제거할 예정

![](https://jihyun416.github.io/assets/aws_13_04.png)

- Route53을 이용하여 연결할 도메인을 대체 도메인 이름에 기입 (여기 작성해야 Route53에서 등록 시 검색이 됨)
- 사용자 정의 SSL 인증서를 등록

![](https://jihyun416.github.io/assets/aws_13_05.png)

- 생성된 배포 내로 접근하여 [오류페이지] 를 세팅
- SPA 특성을 이용하여 어떤 경로일지라도 SPA 인덱스 페이지로 이동시켜준다.

![](https://jihyun416.github.io/assets/aws_13_06.png)

- 403/404에 대해 어플리케이션 메인 페이지로 이동시킴
- HTTP 응답코드는 정상 코드인 200로 변경

![](https://jihyun416.github.io/assets/aws_13_07.png)

여기까지 진행하면 버킷에 정적페이지 빌드 산출물(html, js 등)을 올려두면 CloudFront 주소로 혹은 연결한 도메인으로 접근이 가능하다!



위 까지가 정적 웹호스팅에 관한 것이고



유지보수가 용이하도록

1. 리포지토리에 Push 할 경우 자동으로 배포되도록,

2. 문제가 있을 경우 기존 파일로 롤백할 수 있도록

CodePipeline을 이용하여 추가 구성하였다.



## 3. CodePipeline 설정

### 1) buildspec.yml 정의

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 12
  build:
    commands:
      - npm run build
  post_build:
    commands:
      - aws s3 rm s3://[백업버킷명]/ --recursive
      - aws s3 sync s3://[버킷명] s3://백업버킷명
      - aws s3 rm s3://[버킷명]/ --recursive
      - aws s3 sync dist/ s3://[버킷명]
      - aws cloudfront create-invalidation --distribution-id [CloudFront아이디] --paths "/*"
cache:
  paths:
    - 'node_modules/**/*'
```

- nodejs 환경을 지정한다.
- build 단계에서 빌드 명령어를 수행한다.
- post_build 단계
  - 백업 버킷 파일 삭제
  - 현재 버킷에 있는 파일을 백업버킷으로 복제 (싱크)
  - 현재 버킷에 있는 파일 삭제
  - 빌드한 파일을 현재 버킷으로 복제 (싱크)
  - 전체 경로에 대해 CloudFront 무효화 요청



### 2) buildspec-rollback.yml 정의

```
version: 0.2

phases:
  build:
    commands:
      - aws s3 rm s3://[현재버킷명]/ --recursive
      - aws s3 sync s3://[백업버킷명] s3://[현재버킷명]
      - aws cloudfront create-invalidation --distribution-id [CloudFront아이디] --paths "/*"
```

- build 단계 (빌드는 아니지만~)
  - 현재 버킷 파일 삭제
  - 백업 버킷에 있는 파일을 현재 버킷으로 복제
  - 전체 경로에 대해 CloudFront 무효화 요청



### 3) CodePipeline 생성

![](https://jihyun416.github.io/assets/aws_13_08.png)

1단계 : 소스 (CodeCommit)

2단계 : 빌드 배포 (CodeBuild + buildspec.yml)

3단계 : 롤백 수동승인 (Approval)

4단계 : 롤백 (CodeBuild + buildspec-rollback.yml)

로 구성한다.

배포에 문제가 없을 경우 2단계 까지만 이용하고, 배포에 이상이 있을 경우 3단계 수동승인에서 [승인]을 눌러 4단계 롤백을 이용하여 기존 파일을 덮어쓰도록 구성한다.

CodeBuild 프로젝트는 Pipeline을 통해 생성하고 (소스 단계 이어받기 위함)

프로젝트 IAM 권한에 S3와 CloudFront 접근 권한을 부여해야 한다.
