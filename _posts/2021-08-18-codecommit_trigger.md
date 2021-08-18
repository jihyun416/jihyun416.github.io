---
layout: post
title:  AWS - Codecommit Push Slack 연동
author: Jihyun
category: was
tags:
- aws
- codecommit
- lambda
- aws sns
- aws iam
- aws cloud watch
- slack
- cicd
date: 2021-08-18 18:15 +0900
---

AWS의 Git Service인 CodeCommit에 이벤트 발생 시, Slack에 Webhook을 통해 알림을 전송하자!

**[구현 시 이점]**

1) 협업 동료에게 변경을 알릴 수 있다.
2) 일한 티가 난다🤣

*2020년 4월에 Slack이랑 연동하는 서비스가 나온 것 같지만 고전(?)적인 방식으로*..



![구성도](https://jihyun416.github.io/assets/aws_1_1.png)

- CodeCommit에 모든 리파지토리 이벤트에 대해 SNS에 이벤트를 발행한다.
- Lambda에서 SNS를 구독해서 이벤트가 발생 시 Commit 정보를 추출하여 Slack에 전송한다.



## 1. Amazon SNS - Create Topic

![Create SNS](https://jihyun416.github.io/assets/aws_1_2.png)

- Amazon SNS - Topics - Create topic
  - Type : Standard 로 설정 *(순서가 그리 이슈되지 않기 때문에 fifo를 할필요가 없음*)
  - 다른 부분은 기본값으로 설정



## 2. CodeCommit - Create Trigger

![CodeCommit Trigger](https://jihyun416.github.io/assets/aws_1_3.png)

- CodeCommit - Repositories - [Repository] - Settings - Triggers - Create trigger

  - Events : All repository events 선택 

    *(이전 회사에서 Push to existing branch로 되어있어서 feature 최초 생성 시에는 커밋메시지가 안보이는 이슈가 있었다. 모든 이벤트에 대해 감지하도록 하자!)*

  - Choose the service to use 에서 Amazon SNS 선택 후 위에서 만든 Topic 선택



## 3. AWS Lambda - Create Function

![Create Lambda](https://jihyun416.github.io/assets/aws_1_4.png)

- Python 3.8 선택

- 다른 부분은 기본값으로 설정

![trigger](https://jihyun416.github.io/assets/aws_1_9.png)

![trigger](https://jihyun416.github.io/assets/aws_1_8.png)

- function 생성 후 안에서 상단의 Add trigger를 눌러 위에서 생성한 SNS를 추가한다.



#### 소스 코드

```python
import json
import boto3
from urllib import request

codecommit = boto3.client('codecommit')

def lambda_handler(event, context):
    subject = event['Records'][0]['Sns']['Subject']
    Message = json.loads(event['Records'][0]['Sns']['Message'])
    record = Message['Records'][0]
    deleted =  record['codecommit']['references'][0].get('deleted')
    commitID = record['codecommit']['references'][0]['commit']+""
    
    tmp = record['eventSourceARN'].split(":")
    repository = tmp[len(tmp)-1]

    tmp = record['codecommit']['references'][0]['ref'].split("/")
    branchName = tmp[len(tmp)-1]

    tmp = Message['Records'][0]['userIdentityARN'].split(":")
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
elif branchName == "master":
    color = "#FF0000"

channel = "slack-channel-name"

title="USER :" + user+ "\nBranch : " + branchName+ "\n Commit ID : " + commitID + "\n Commit Message : "+ commitMessage + "\n Commit file : "+ diff

if deleted:
    title="Delete Branch>>>>\n"+title

send_slack_message({
        'channel': channel,
        'text': subject,
        'attachments': [
            {
                "title": title,
                "color": color
            },
            {
                "title": "CodeCommit Commit Link",
                "title_link": commitURL,
                "color": color
            }
        ]
    })

return {
    'statusCode': 200,
    'body': json.dumps('CodeCommit Trigger!')
}

def send_slack_message(slack_message) :
    HOOK_URL =  "webhook url"
    req = request.Request(HOOK_URL, json.dumps(slack_message).encode("utf-8"),method="POST")
    try:
        response = request.urlopen(req)
        response.read()
        print("Message posted to %s", slack_message['channel'])
    except request.HTTPError as e:
        print("Request failed: %d %s", e.code, e.reason)
    except request.URLError as e:
        print("Server connection failed: %s", e.reason)
```

- Lambda의 실행 로그는 CloudWatch - Log - Log groups 에서 Lambda 함수명으로 찾으면 볼 수 있다.
- Lambda parameter인 event를 찍어보면 SNS로부터 받은 Json 형태의 이벤트 정보를 볼 수 있는데, 이 이벤트를 파헤쳐 보면 여러 정보들을 얻어낼 수 있다.
  - commit id
  - branch name
  - User
- CodeCommit - Repositories - Commits 에서 볼 수 있는 커밋 상세 내역은 일정한 패턴을 가지고 있는데, 이를 활용하여 커밋 상세 URL 정보를 줄 수 있다.
- Lambda 함수가 사용하는 IAM Roles에 CodeCommit 관련한 권한이 있어야 boto3를 통한 Codecommit 접근이 가능하다. 정보를 읽기만 할 것이므로 AWSCodeCommitReadOnly 권한을 추가하였다.

![](https://jihyun416.github.io/assets/aws_1_7.png)

- **Boto3** 라이브러리를 이용하여 위에서 얻어낸 repository와 commit id를 기준으로 커밋 상세정보를 가져올 수 있다.
  - **get_commit**을 통해 상세 정보 가져오기
  - 위에서 가져온 정보를 토대로 부모 commit id를 확인하여 **get_differences**를 이용하여 변동 파일 추출
- Slack Webhook 가이드를 참고하여 Http Post 요청으로 Webhook 전송

![](https://jihyun416.github.io/assets/aws_1_6.png)



#### 참고

> 나의 AWS 사수(?) David Ryu님께서 남겨주신 유산 (잘 지내시죠?)
