---
layout: post
title:  AWS - Client VPN
author: Jihyun
category: AWS
tags:
- aws
- aws client vpn endpoint
- aws client vpn
- vpn
date: 2021-12-01 12:20 +0900
---

보안을 유지하기 위해서는 외부와 직접 통신하지 않는 리소스들은 **Private subnet**에 두는 것이 좋다.

하지만 이렇게 하면 외부의 접근을 막을 수 있지만 개발자도 못들어가는 문제가 있는데,

SSH만 사용할 경우 Bastion host 등의 방법으로 우회 접근이 가능하지만, 

DB의 경우는 DB Tool로도 접근이 가능해야 하며, 로컬에서 개발 중일 때도 DB에 접근이 가능해야 한다.

또한 Elasticache, DynamoDB 와 같이 VPC내에서만 접근 가능한 완전관리형 서비스 들도 있다.

따라서 **AWS Client VPN**을 이용하여 개발자 PC가 VPC 상에 있는 것처럼 만들어 리소스에 엑세스 할 것이다.



## 1. 인증서 생성

상호 인증 방식을 쓰기 위해 아래와 같은 절차로 인증서를 생성한다.

### 1) OpenVPN easy-rsa clone, 새 PKI 환경 시작

```shell
$ git clone https://github.com/OpenVPN/easy-rsa.git
$ cd easy-rsa/easyrsa3
$ ./easyrsa build-ca nopass
```

### 2) 새 CA(인증기관) 빌드

```shell
$ ./easyrsa build-ca nopass
```

![build-ca](https://jihyun416.github.io/assets/aws_8_1.png)

- 위와 같이 명령하면 Common Name 을 묻는다. 적당한 이름을 입력하고 엔터치면 ca가 생성된다.

### 3) 서버 인증서 및 키 생성

```shell
$ ./easyrsa build-server-full server nopass
```

### 4) CA와 서버 인증서 및 키를 한 폴더에 모은 뒤 이동

```shell
$ mkdir ~/acm_upload/
$ cp pki/ca.crt ~/custom_folder/
$ cp pki/issued/server.crt ~/custom_folder/
$ cp pki/private/server.key ~/custom_folder/
$ cd ~/acm_upload/
```

### 5) ACM에 서버 인증서 업로드

```shell
$ aws acm import-certificate --certificate fileb://server.crt --private-key fileb://server.key --certificate-chain fileb://ca.crt
```

### 6. Certificate Manager에 잘 등록되었는지 확인

![acm](https://jihyun416.github.io/assets/aws_8_2.png)



## 2. 클라이언트 VPC 엔드포인트 생성

![](https://jihyun416.github.io/assets/aws_8_3.png)

- 할당 받을 IP CIDR 입력한다.
  - 서브넷의 CIDR 블록은 Client VPN 엔드포인트의 클라이언트 CIDR와 겹칠 수 없으니 연결하려는 VPC와 완전히 다른 영역으로 생성한다.
- [상호인증 사용]을 선택하고 위에서 생성하여 업로드한 acm을 서버와 클라이언트 동일하게 선택한다.

![](https://jihyun416.github.io/assets/aws_8_4.png)

- 분할터널을 활성화 한다. (Client VPN 엔드포인트 라우팅 테이블의 경로와 일치하는 네트워크 대상 트래픽만 Client VPN 터널을 통해 라우팅)

  > - 목적지가 AWS인 트래픽만을 VPN 터널을 통과하도록 하여 클라이언트의 트래픽 라우팅을 최적화할 수 있습니다.
  > - AWS에서 송신하는 트래픽 양을 줄일 수 있고, 이에 따라 데이터 전송 비용을 절감할 수 있습니다.

- VPN 포트는 1194로 선택 

  - 443과 1194중 선택할 수 있는데, 443으로 했을 때 client 연결 시 port 충돌 현상이 있었음



### 연결 탭

![](https://jihyun416.github.io/assets/aws_8_5.png)

![](https://jihyun416.github.io/assets/aws_8_6.png)

- 모든 가용 영역이 연결될 수 있도록 서브넷 선택 (프라이빗 서브넷 2개를 연결함)
  - 한 Client VPN Endpoint는 같은 VPC내에 있는 서브넷들만 연결 가능하다. 따라서 VPC가 여러개인데 VPN이 필요한 경우 각각 생성해줘야 한다.



### 라우팅 테이블

![](https://jihyun416.github.io/assets/aws_8_7.png)

- 연결에 서브넷을 등록하면 기본적으로 VPC CIDR와 동일하게 대상 CIDR가 등록된다.
  - 클라이언트에게 인터넷 엑세스를 허용하려면, 각 서브넷에 대해 0.0.0.0/0 라우팅을 추가하라고 하는데, 분할터널을 활성화 했으니 추가하면 오히려 모든 트래픽이 vpn을 타게 되므로 추가를 안하는 것이 맞는듯 하다.



## 3. 클라이언트 ovpn 생성

이제 개발자의 PC에서 vpn client 프로그램을 이용하여 vpn에 접속하면 되는데, 접속을 하기 위해서는 *.ovpn 파일이 필요하다.

파일은 개인별로 발행하여 개인별로 각각의 파일로 접근하도록 한다.

### 1) 클라이언트 구성 다운로드

![](https://jihyun416.github.io/assets/aws_8_10.png)

클라이언트 구성을 다운로드 하면 downloaded-client-config.ovpn 파일을 내려받을 수 있다.

![](https://jihyun416.github.io/assets/aws_8_11.png)

파일을 열어보면 상단에 vpn endpoint에 대한 정보가 있고 <ca></ca> 가 확인된다.

이 파일을 개인별로 복사한 뒤 클라이언트 인증서 및 키를 생성하여 파일 내에 <cert></cert> <key></key>를 추가해주면 된다.

### 2) 클라이언트 인증서 및 키 생성

```shell
$ cd easy-rsa/easyrsa3
$ ./easyrsa build-client-full [사용자이름] nopass
```

- 아까 서버 인증서를 만들때 생성한 easyrsa3 폴더로 이동한 뒤 클라이언트 인증서/키를 생성한다. (아래와 같이 파일이 생성됨)
  - easy-rsa/easyrsa3/pki/issued/[사용자이름].crt
  - easy-rsa/easyrsa3/pki/private/[사용자이름.]key

- ovpn파일을 사용자명.ovpn 파일로 변경한 뒤 편집한다.
  - <ca></ca> 아래 <cert></cert> 태그를 추가하고 [사용자이름].crt 파일 내에 -----BEGIN CERTIFICATE----- -----END CERTIFICATE----- 부분을 복사하여 넣는다.
  - 그 아래 <key></key> 태그를 추가하고 [사용자이름].key 파일 내용을 복사하여 넣는다.

- 사용자별로 ovpn 파일을 전달한다.

### .ovpn 파일 구조

```
client
dev tun
proto udp
remote cvpn-endpoint-xxxx.amazonaws.com 1194
remote-random-hostname
resolv-retry infinite
nobind
remote-cert-tls server
cipher AES-256-GCM
verb 3
<ca>
-----BEGIN CERTIFICATE-----
MIIDJDCCAgygA....
-----END CERTIFICATE-----

</ca>

<cert>
-----BEGIN CERTIFICATE-----
MIIDPjCCA....
-----END CERTIFICATE-----
</cert>

<key>
-----BEGIN PRIVATE KEY-----
MIIEvQI......
-----END PRIVATE KEY-----
</key>

reneg-sec 0
```



## 4. VPN 접속

각자 환경에 적절한 VPN Client 프로그램을 다운 받아 ovpn 파일을 등록 후 연결하면 된다.

VPC private subnet에 있는 RDS를 접속하여 VPN이 정상적으로 연결되었는지 검증해보았다. (잘된다!)

무료로 사용할 수 있는 여러가지 프로그램이 있는데, Mac M1 기준으로 [Tunnelblick](https://tunnelblick.net/)이 나름 괜찮은 것 같다. (AWS Client VPN은 접속 종료가 좀 이상하게 작동함..)



## 5. 클라이언트 인증서 취소하기

클라이언트 인증서 해지 목록을 사용하여 특정 클라이언트 인증서의 Client VPN 엔드포인트에 대한 액세스를 취소할 수 있다.

```shell
$ cd easy-rsa/easyrsa3
$ ./easyrsa revoke [사용자이름]
```

- 위와 같이 명령하면 계속 할건지 물어보는 창이 뜨고 yes 라고 입력하면 계속된다.

  ![](https://jihyun416.github.io/assets/aws_8_12.png)

- 완료 시 생성된 pem 파일의 경로가 출력된다. 예) CRL file: /Users/jessy/easy-rsa/easyrsa3/pki/crl.pem

```
aws ec2 import-client-vpn-client-certificate-revocation-list --certificate-revocation-list file://[crl파일절대경로입력] --client-vpn-endpoint-id [client vpn endpoint id] --region [region]
```

- 위와 같이 생성한 파일을 내보내면 취소가 완료된다.

  - 콘솔에서 pem을 복사해서 내보낼 수 있다.

    ![](https://jihyun416.github.io/assets/aws_8_13.png)

- 취소한 ovpn으로 접속을 시도하면 접속이 안되는 것을 확인할 수 있다.

  ![](https://jihyun416.github.io/assets/aws_8_14.png)



#### 참고

>[무스마 기술블로그 | AWS Client VPN Endpoint 사용하기](https://musma.github.io/2019/11/04/aws-client-vpn-endpoint.html)
>
>[AWS Documentation - AWS Client VPN - 라우팅](https://docs.aws.amazon.com/ko_kr/vpn/latest/clientvpn-admin/cvpn-working-routes.html)
>
>[AWS Documentation - AWS Client VPN - 인증](https://docs.aws.amazon.com/ko_kr/vpn/latest/clientvpn-admin/client-authentication.html)
>
>[AWS Documentation - AWS Client VPN - 클라이언트 인증서 해지](https://docs.aws.amazon.com/ko_kr/vpn/latest/clientvpn-admin/cvpn-working-certificates.html)





