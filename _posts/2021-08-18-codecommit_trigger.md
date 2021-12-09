---
layout: post
title:  AWS - Codecommit Push Slack 연동
author: Jihyun
category: aws
tags:
- aws
- codecommit
- lambda
- aws iam
- aws cloud watch
- slack
- cicd
date: 2021-08-18 18:15 +0900
last_modified_at: 2021-12-09 12:30:00 +0900
---

*변경 내용 : CodeCommit -> SNS -> Lambda 형태에서 CodeCommit -> Lambda 형태로 변경*



AWS의 Git Service인 CodeCommit에 이벤트 발생 시, Slack에 Webhook을 통해 알림을 전송하자!

**[구현 시 이점]**

1) 협업 동료에게 변경을 알릴 수 있다.
2) 일한 티가 난다🤣

CodeCommit에서 바로 슬랙으로 알림을 추가할 수 있지만, Lambda를 이용하여 직접 구현하여 커밋 내용을 보여주고 CodeCommit으로 링크를 제공할 것이다.

![](https://jihyun416.github.io/assets/aws_11_1.png)

- CodeCommit에 모든 리파지토리 이벤트에 대해 SNS에 이벤트를 발행한다.
- Lambda에서 SNS를 구독해서 이벤트가 발생 시 Commit 정보를 추출하여 Slack에 전송한다.



## 1. Lambda 생성

#### 소스 코드

```python
import json
import boto3
from urllib import request

codecommit = boto3.client('codecommit')

def lambda_handler(event, context):
    record = event['Records'][0]
    deleted =  record['codecommit']['references'][0].get('deleted')
    commitID = record['codecommit']['references'][0]['commit']
    tmp = record['eventSourceARN'].split(":")
    repository = tmp[len(tmp)-1]
    tmp = record['codecommit']['references'][0]['ref'].split("/")
    branchName = tmp[len(tmp)-1]
    tmp = record['userIdentityARN'].split(":")
    user = tmp[len(tmp)-1].replace("user/","")
    commitURL = "https://ap-northeast-2.console.aws.amazon.com/codesuite/codecommit/repositories/" + repository + "/commit/"+commitID+"?region=ap-northeast-2"
    commitInfo = codecommit.get_commit(repositoryName=repository,commitId=commitID)
    commitMessage = commitInfo['commit']['message']
    
    diff = ""
    for parentCommitId in commitInfo['commit']['parents']:
        commitDifferences = codecommit.get_differences(repositoryName=repository,beforeCommitSpecifier=parentCommitId, afterCommitSpecifier=commitID)
        for difference in commitDifferences['differences'] :
            if difference['changeType'] == 'D':
                diff = diff + difference['beforeBlob']['path'] + " - Delete \n"
            else :
                diff = diff + difference['afterBlob']['path'] + "\n"
                
    color = "#5F00FF"
    if branchName == "develop":
        color = "#FFE400"
    elif branchName == "release":
        color = "#FF0000"
        
    content="USER :" + user+ "\nBranch : " + branchName+ "\n Commit ID : " + commitID + "\n Commit Message : "+ commitMessage + "\n Commit file : "+ diff
    if deleted:
        content="Delete Branch>>>>\n"+content
    
    slack_message = {
        "icon_emoji":":technologist:",
        "blocks": [
        	{
        		"type": "section",
        		"block_id": "section1",
        		"text": {
        			"type": "mrkdwn",
        			"text": ":mega:<"+commitURL+"|* AWS CodeCommit Notification*>"
        		}
        	}
        ],
        'attachments': [
            {
                "title": content,
                "color": color
            }
        ]
    }
    
    HOOK_URL =  "Webhook 주소"
    req = request.Request(HOOK_URL, json.dumps(slack_message).encode("utf-8"),method="POST")
    try:
        response = request.urlopen(req)
        response.read()
        print("Message posted")
    except request.HTTPError as e:
        print("Request failed: %d %s", e.code, e.reason)
    except request.URLError as e:
        print("Server connection failed: %s", e.reason)
    
    return {
        'statusCode': 200,
        'body': json.dumps('CodeCommit Trigger!')
    }
```

- Lambda의 실행 로그는 CloudWatch - Log - Log groups 에서 Lambda 함수명으로 찾으면 볼 수 있다.
- Lambda parameter인 event를 찍어보면 Json 형태의 이벤트 정보를 볼 수 있는데, 이 이벤트를 파헤쳐 보면 여러 정보들을 얻어낼 수 있다.
  - commit id
  - branch name
  - User
- CodeCommit - Repositories - Commits 에서 볼 수 있는 커밋 상세 내역은 일정한 패턴을 가지고 있는데, 이를 활용하여 커밋 상세 URL 정보를 줄 수 있다.



## 2. Lambda 트리거 추가

![](https://jihyun416.github.io/assets/aws_11_2.png)

![](https://jihyun416.github.io/assets/aws_11_3.png)

- Lambda의 트리거 추가를 통해 CodeCommit 트리거를 추가한다.
  - 여기서 추가하면 CodeCommit에서 Lambda 함수를 실행할 수 있는 권한이 자동 설정된다. (**구성-권한** 에서 확인 가능)
- 다른 리포지토리도 이 람다를 사용하려면 트리거 추가를 통해 추가해주면 된다.



## 3. Lambda가 사용하는 IAM Role에 CodeCommit 권한 추가

Lambda 함수가 사용하는 IAM Roles에 CodeCommit 관련한 권한이 있어야 boto3를 통한 Codecommit 접근이 가능하다. 

정보를 읽기만 할 것이므로 AWSCodeCommitReadOnly 권한을 추가한다.

![](https://jihyun416.github.io/assets/aws_11_4.png)

- **Boto3** 라이브러리를 이용하여 위에서 얻어낸 repository와 commit id를 기준으로 커밋 상세정보를 가져올 수 있다.
  - **get_commit**을 통해 상세 정보 가져오기
  - 위에서 가져온 정보를 토대로 부모 commit id를 확인하여 **get_differences**를 이용하여 변동 파일 추출
- Slack Webhook 가이드를 참고하여 Http Post 요청으로 Webhook 전송

![](https://jihyun416.github.io/assets/aws_11_5.png)



#### 참고

> 나의 AWS 사수(?) David Ryu님께서 남겨주신 유산 참고 (잘 지내시죠?)