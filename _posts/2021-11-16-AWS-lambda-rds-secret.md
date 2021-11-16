---
layout: post
title:  AWS - Lambda, RDS Proxy, Secrets Manager
author: Jihyun
category: AWS
tags:
- aws
- lambda
- rds proxy
- secrets manager
date: 2021-11-16 17:30 +0900
---

Lambda를 이용하여 RDS에 접근하는 프로그램을 만들 것이다.

Host와 User/Password만 안다면 DB에 직접 접근하는 것도 가능하지만 직접 접근할 경우 항상 Cool start가 이루어지기 때문에 이를 방지하기 위해 RDS Proxy를 이용할 것이다.

> RDS Proxy helps you manage a large number of connections from **Lambda** to an RDS database by establishing a warm connection pool to the database. Your Lambda functions can scale to meet your needs and use the RDS Proxy to serve multiple concurrent application requests.

![](https://jihyun416.github.io/assets/aws_7_1.png)

사용 서비스

- RDS
- RDS Proxy
- Secrets Manager - RDS 보안 암호 관리
- Lambda 
- AWS IAM - RDS Proxy 접근 시 암호가 아닌 IAM 인증을 사용



## 1. RDS user 생성

다른 어플리케이션과 별개로 서버리스 서비스에서 접근할 때 사용할 사용자를 새로 생성할 것이다.

어플리케이션과 사용자를 공유하지 않는 이유는 접근 주체에 따라 사용자를 만들어서 해야 하는 이유도 있지만 Secrets Manager 사용의 이점을 살리기 위해서이다.

Secrets Manager에서는 보안 암호를 자동으로 교체할 수 있는 옵션이 있는데, Secrets Manager를 사용하지 않는 어플리케이션이 동일한 사용자를 사용하고 있다면, 암호가 자동으로 바뀐다면 낭패를 볼 수 있다.

보안 유지를 위해 자동 교체 옵션을 사용하기 위해 사용자를 분리할 것이다.

```sql
use mysql;
select host, user, authentication_string from user;
show grants for 'admin'@'%';
create user 'serverless'@'%' identified by '암호입력';
show grants for 'serverless'@'%';
grant SELECT, INSERT, UPDATE, DELETE on *.* TO 'serverless'@'%'
```

- mysql 스키마로 전환한다
- 현재 사용자 목록을 확인한다
- 현재 사용하고 있는 admin이 무슨 권한을 가지고 있는지 체크한다.
  - GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ... CREATE USER, EVENT, TRIGGER ON *.* TO `admin`@`%` WITH GRANT OPTION
    - CREATE USER 권한이 있다.
    - 여러 자격을 가지고 있으며 WITH GRANT OPTION이 있어 가지고 있는 자격을 다른 유저에게 줄 수 있다.
- 사용자를 비밀번호와 함께 생성한다.
- 현재 권한을 확인한다.
- 단순 DML 권한을 부여한다.



## 2. Secrets Manager에 새 보안암호 생성

RDS Proxy는 무조건 Secrets Manager를 이용하여 인증해야 한다. 이용할 자격 증명을 Secrets Manager에 새로 등록한다.

![](https://jihyun416.github.io/assets/aws_7_2.png)

- Amazon RDS에 대한 자격 증명을 선택하고 위에서 만든 user와 암호를 입력한다.
- 해당 RDS를 선택한다.

![](https://jihyun416.github.io/assets/aws_7_3.png)

- 보안암호 이름을 지정한다. (username만 쓰지 말고 어떤 보안암호인지 알아볼 수 있도록 쓰자!)

![](https://jihyun416.github.io/assets/aws_7_4.png)

- 보안암호가 자동으로 교체되도록 활성화한다.
- 교체함수를 생성하여 새롭게 Lambda 교체 함수가 생기도록 지정한다. (CloudFormation으로 자동 생성됨)

![](https://jihyun416.github.io/assets/aws_7_5.png)

![](https://jihyun416.github.io/assets/aws_7_6.png)

![](https://jihyun416.github.io/assets/aws_7_7.png)

- 보안 암호가 등록되었으며, 교체함수에 의해 바로 암호가 한번 교체된다.

![](https://jihyun416.github.io/assets/aws_7_8.png)

- 아래에 보면 각 언어별로 SDK를 이용하여 어떻게 secret 정보를 가져올 수 있는지 샘플 코드가 있다.
- RDS Proxy에서 이를 직접적으로 활용하지는 않지만 어플리케이션에서 암호를 불러올 필요가 있을 때 이를 활용하면 된다.
  - RDS 뿐만 아니라 보안에 필요한 요소를 Secrets Manager에 저장하여 호출하여 활용하는 것도 좋을듯!!

![](https://jihyun416.github.io/assets/aws_7_9.png)

- 자동 생성된 교체함수 CloudFormation 스택이다.



## 3. RDS Proxy 생성

![](https://jihyun416.github.io/assets/aws_7_10.png)

- 프록시 이름을 입력한다. (RDS이름 + _proxy 와 같은 형식으로 입력했음)
- RDS MySQL을 쓰고 있기 때문에 엔진호환성은 MySQL 체크 (PostgreSQL도 지원한다!)
- IAM인증을 사용할 것이기 때문에 전송 계층 보안 필요도 체크한다

![](https://jihyun416.github.io/assets/aws_7_11.png)

- 프록시와 연결할 RDS를 선택한다
- 연결 풀은 최대 100%까지 선택 가능하나, 다른 어플리케이션에서 사용을 하고 있는 관계로 30%만 사용하는걸로 설정하였다.

![](https://jihyun416.github.io/assets/aws_7_13.png)

- 위에서 생성한 Secrets Manager 보안 암호를 선택하여 추가한다. (여러개 추가 가능)
- IAM은 자동 생성되도록 IAM 역할 생성을 선택한다.
- IAM을 통해 인증하기 위해 IAM 인증을 활성화한다.

![](https://jihyun416.github.io/assets/aws_7_14.png)

- 생성이 완료되면 프록시 앤드포인트를 확인할 수 있다. 이 앤드포인트를 RDS 앤드포인트를 대신하여 사용하면 프록시를 이용한 접속이 가능하다.



## 4. Lambda IAM 설정

사용할 람다의 IAM에 RDS Proxy 접근 및 VPC 접근과 관련하여 정책 추가가 필요하다.

람다의 구성에서 추가하면 손쉽게 IAM에 정책을 추가할 수 있다.

![](https://jihyun416.github.io/assets/aws_7_15.png)

![](https://jihyun416.github.io/assets/aws_7_16.png)

- 위에서 생성한 프록시 연결
- 여기서 추가한다고 딱히 람다 소스에서 데이터베이스 프록시에 접근하는데 도움이 되는것은 아님
  - IAM 정책 추가 역할
  - 이 람다에서 이 데이터베이스 프록시를 쓴다는 명시적인 역할
  - 데이터베이스 프록시 쓰려면 VPC에 함수를 연결해야 한다는 알림이 나옴

![](https://jihyun416.github.io/assets/aws_7_17.png)

![](https://jihyun416.github.io/assets/aws_7_18.png)

- 람다 함수를 VPC에 연결, RDS Proxy와 같은 VPC여야 한다.
  - *지금 딱히 VPC, 서브넷, 보안그룹을 디테일하게 관리를 안한 상태인데 세분화에서 관리를 해야할듯...*

![](https://jihyun416.github.io/assets/aws_7_19.png)

- 람다의 역할 IAM에 들어가보면 정책이 3가지 있는 것을 확인할 수 있다.
  - AWSLambdaBasicExecutionRole- : 람다 최초 생성 시 추가되는 정책
  - AWSLambdaRDSProxyExecutionRole- : RDSProxy에 접근하기 위한 Role, 람다 구성에서 데이터베이스 프록시를 추가하면 자동 생성된다.
  - AWSLambdaVPCAccessExecutionRole- : VPC에 접근하기 위한 Role, 람다 구성에서 VPC 등록 시 자동 생성된다.

![](https://jihyun416.github.io/assets/aws_7_20.png)

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "rds-db:connect",
            "Resource": "arn:aws:rds-db:*:416849462746:dbuser:*/*"
        }
    ]
}
```

- lambda에서 RDSProxy에 접근할 때 유저/패스워드가 아닌 IAM을 이용하기로 했기 때문에 IAM이 DB에 connect할 수 있도로 정책 추가가 필요하다.
- 위와 같은 내용으로 이름을 정해서 위와 같이 정책(policy)를 생성한다.

![](https://jihyun416.github.io/assets/aws_7_21.png)

- 람다의 IAM에 위에서 생성한 정책을 연결한다.



## 5. Lambda 함수

![](https://jihyun416.github.io/assets/aws_7_22.png)

- 소스에서 사용할 환경변수를 구성-환경변수에서 등록한다.

![](https://jihyun416.github.io/assets/aws_7_23.png)

- RDS pem 파일을 업로드를 통해 추가한다. (SSL)
  - https://s3.amazonaws.com/rds-downloads/rds-combined-ca-bundle.pem

```python
import json
import boto3
from os import environ
import pymysql
import ssl

def lambda_handler(event, context):
    region = environ.get('IN_REGION')
    host = environ.get('IN_HOST')
    port = environ.get('IN_PORT')
    username = environ.get('IN_USERNAME')
    database = environ.get('IN_DATABASE')
    client = boto3.Session().client('rds')
    token = client.generate_db_auth_token(
        DBHostname=host,
        Port=port,
        DBUsername=username,
        Region=region
    )
    ctx = ssl.create_default_context(cafile='./rds-combined-ca-bundle.pem')
    ctx.check_hostname = False
    ctx.verify_mode = ssl.CERT_NONE
    connection = pymysql.connect(
        host=host,
        user=username,
        password=token,
        db=database,
        charset='utf8mb4',
        cursorclass=pymysql.cursors.DictCursor,
        ssl=ctx
    )
    cursor = connection.cursor()
query = "SELECT id from my_table"
cursor.execute(query)
results = cursor.fetchmany(3)

return {
    'statusCode': 200,
    'body': json.dumps(results)
}
```



#### 참고

>[[RDS] MySQL 사용자 추가, 데이터 베이스 생성, 권한 부여](https://dev-elena-k.tistory.com/13)
>
>[[AWS] 서버리스를 위한 RDS Proxy서비스](https://medium.com/harrythegreat/aws-%EC%84%9C%EB%B2%84%EB%A6%AC%EC%8A%A4%EB%A5%BC-%EC%9C%84%ED%95%9C-rds-proxy%EC%84%9C%EB%B9%84%EC%8A%A4-fb5815b83cce)
>
>[Stack Overflow - SSL Certificate Verify Failed](https://github.com/MagicStack/asyncpg/issues/238)
>
>[Youtube - Introduction to RDS Proxy](https://www.youtube.com/watch?v=ULRnn6tIYu8)
>
>[Amazon RDS Proxy 집중 탐구 - 윤석찬 :: AWS Unboxing 온라인 세미나](https://www.youtube.com/watch?v=Rq4I57eqIp4)
>
>[IAM 데이터베이스 액세스를 위한 IAM 정책 생성 및 사용](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.IAMPolicy.html)





