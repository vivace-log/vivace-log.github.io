---
title: Nginx와 reverse proxy # 파일 명은 영어만 써야하지만, 여기는 한글이 가능
author: smpringles24 # \_data/authors.yml 에 있는 author id (여러명인경우 authors: [id1, id2, ...])
date: 2023-11-06 20:00:00 +0900 # +0900은 한국의 타임존  (날짜가 미래인경우 빌드시 스킵함.)
categories: [network, nginx] # 카테고리는 메인, 서브 2개만 설정 가능 (띄어쓰기 가능)
tags: [nginx, reverse proxy, newtwork, apache] # 태그는 개수제한 X (띄어쓰기 가능)
# toc: true # Table Of Content는 기본적으로 true로 되어있음.
# comments: true # 기본적으로 true
# pin: false # 글 고정 여부로 기본적으로 false, 공지 같은게 아니라면 그대로 두기
# math: false # 성능 문제로 기본적으로 false로 되어있음. 수식을 적는다면 true로 바꿀 것
# mermaid: false # 다이어그램 툴로써 성능 문제로 기본적으로 flase로 되어있음.  https://mermaid.js.org/intro/
render_with_liquid: false
---

# 🤔발표 주제선정

군대 개발 발표 소모임 Vivache의 첫번째 발표로 내용을 찾던 도중, 자대에 처음 와서 개발을 하기 위해 구축했던 개발 환경 세팅이 떠올랐다. 자대에 처음 왔을 당시, 사지방에서 원하는 퍼포먼스로 개발을 하기에는 여러 불편사항들이 있었다. 예를 들어 파일 다운로드도 자유롭지 못하고, 브라우저 접속도 http/https이외의 프로토콜이 제한되는 등의 문제가 있었다. 이를 해결하기 위해 집에 NAS 서버를 들이고, 홈 네트워크를 구축하고 아파치 과카몰리라는 시스템으로 집에 있는 pc에 원격접속을 시도했고 결과적으로 만족스러운 개발 환경을 갖출 수 있었다.

이때 네트워크에 대해 공부하고, 특히 ip, DNS, DDNS, Reverse Proxy같은 개념을 습득했었는데, 대략 6개월 정도가 지난 지금, 그 당시에 익힌 지식들을 정리, 보완하여 발표를 해보려고 한다. 

# 📓1.1. Apache web server

1998년 간단한 웹 서버 배포를 위해 apache web server라는 툴이 공개되었다. apache web server는 '모듈'이라는 개념을 적용하여 타사의 DBMS, 여러 프로토콜 등을 툴에 적용할 수 있게 하였다. 이는 확장성을 크게 넓혀주었고, apache web server의 사용자는 점점 늘어났다.

그런데 2000년대를 지나면서 서서히 일반 가정에도 컴퓨터와 휴대전화가 보급되기 시작하면서 apache web server에 문제가 생기기 시작한다. apache web server는 클라이언트와 서버가 1대1로 connect를 맺고 응답을 주고받기 때문에 client수가 증가할수록 그대로 서버쪽의 부담이 증가하게 된다. 이 문제는 클라이언트의 절대적인 수가 증가함에 따라 점점 거론되기 시작했고 이는 1만개의 connection을 처리해낼 수 없다는 의미의  C10K(connection 10000 problem) 문제로 남았다.


# 📓1.2. Nginx의 등장

2004년, apache web server의 C10K문제를 해결하기 위한 대한으로 Nginx라는 툴이 등장했다. Nginx는 기본적으로 apache web server와 함께 동작하고 apache web server로 가야 할 클라이언트의 요청을 Nginx에서 1차적으로 처리한다. 즉 클라이언트의 요청을 Nginx가 먼저 받아 정적 페이지 처리, 캐시 데이터 반환 등의 간단한 처리를 시도하고 동적 페이지 처리 등의 복잡한 업무만 apache web server에 요청하는 형태이다.

# 📓1.3. Apache web server 와 Nginx의 작동 방식

apache web server에는 10,000개 상당의 클라이언트의 요청을 감당할 수 없는 C10K문제가 있었다. 그렇다면 Nginx에도 10,000개 상당의 클라이언트 요청이 들어오게 되면 문제가 되지는 않을까? 2000년대 초반에 비해 인터넷 사용자는 수십 수백배 증가했을텐데, 어떻게 Nginx는 아직까지 사용될 수 있을까? 그 답은 Nginx와 apache web server의 근본적인 동작 방식의 차이에 있다.

### Apache web server

![apache_work_flow](/assets/img/2023-11-06-nginx_reverse_proxy/apache_work_flow.png)
apche web server에서는 모든 클라이언트의 요청에 connection을 형성한다. 이때 생성된 process들은 cpu의 각 코어에 할당되며, 일반적으로 코어 하나에서 여러개의 process를 책임지게 된다.

### Nginx

![nginx_work_flow](/assets/img/2023-11-06-nginx_reverse_proxy/nginx_work_flow.png)
nginx는 master process와 worker process로 구성된다. master process는 worker process를 관리하는데 사용되며 cpu의 각 core당 일반적으로 하나의 커다란 worker process가 실행되고, 각  worker process는  비동기 이벤트기반 모델을 사용하여 다수의 테스크를 큐 방식으로 처리한다.



nginx의 동작 구조상 얻을 수 있는 이득은 다음과 같다.

- cpu에서 프로세스를 처리할 때, 하나의 프로세스를 처리하고 다른 프로세스를 작업 영역에 올리는 시간이 발생하는데 (context switch) 이러한 비용이 들지 않는다.
- 클라이언트의 요청의 수에 비례해 서버의 리소스 사용이 증가하는 apache web server에 비해 nginx는 실행되는 worker process의 수가 정해져 있기 때문에 리소스 사용량이 비교적 일정하게 유지된다.




이외에도 nginx에서는 성능 향상을 위해 여러 다른 개념이 적용되어있다.

**I/O Non-Blocking :** 비교적 시간이 오래 소요되는 네트워크 I/O 작업이 필요할 때, worker process가 비동기적으로 다른 작업을 동시에 처리하는 방식

**쓰레드 풀(Nginx 1.7.11부터 적용) :** 시간이 많이 걸리는 특수한 작업들을 별도의 쓰레드 풀에서 비동기적으로 처리




# 📓2. 프록시, 리버스 프록시
### 프록시 (Proxy)

![proxy](/assets/img/2023-11-06-nginx_reverse_proxy/proxy.png)

프록시 서버는 클라이언트와 서버 사이에 위치하며 클라이언트의 요청을 서버에 대신 전달한다.

프록시의 장점
- **익명성 보장:** 클라이언트의 실제 IP 주소를 숨기고 프록시 서버의 IP 주소를 사용하여 인터넷을 탐색
- **콘텐츠 필터링:** 웹 트래픽을 모니터링하고 특정 사이트나 콘텐츠에 대한 접근을 차단
- **캐싱:** 자주 요청되는 콘텐츠를 프록시 서버에 저장해둔 뒤, 서버를 거치지 않고 바로 클라이언트에게 응답을 주어 성능을 향상


### 리버스 프록시 (Reverse Proxy)

![reverse_proxy](/assets/img/2023-11-06-nginx_reverse_proxy/reverse_proxy.png)

리버스 프록시는 서버와 클라이언트 사이에 위치하며, 주로 서버 측에서 클라이언트의 요청을 받아 서버로 전달한다.

리버스 프록시의 장점
- **보안 강화:** 원본 서버의 실제 IP 주소를 숨기고, 외부 공격으로부터 보호
- **부하 분산:** 여러 서버로 트래픽을 분산시켜 서버 부하를 줄이고 서비스의 가용성을 향상
- **캐싱 및 압축:** 자주 요청되는 콘텐츠를 프록시 서버에 저장하여 빠르게 제공하고, 데이터를 압축하여 전송 효율을 높임
- **SSL 암호화:** 클라이언트와 리버스 프록시 사이의 통신을 암호화하여 보안을 강화
   
   
   
# 💻구현

그렇다면 nginx에서 reverse proxy를 구현해보자.

#### 1. nginx 설치 (ubuntu 기준)

`$ sudo apt install nginx`

#### 2. nginx 실행

- nginx 실행
`$ sudo systemctl start nginx`

- 시스템 재부팅시 nginx 자동실행
`$ sudo systemctl enable nginx`

#### 3. nginx 파일 구성 살펴보기

![nginx_file](/assets/img/2023-11-06-nginx_reverse_proxy/nginx_file.png)

![nginx_file2](/assets/img/2023-11-06-nginx_reverse_proxy/nginx_file2.png)

일반적으로 사용되는 파일/폴더는 다음과 같다.
nginx.conf : nginx의 메인 config파일. sites-enabled내의 파일들을 참조하여 로드한다.
sites enabled : sites available에 만든 config파일 중 활성화할 파일들을 심볼릭 링크해온다.
sites available : nginx의 서브 config파일을 저장해둔 디렉토리. 활성화되지 않은 상태.

#### 4. nginx.conf 파일 수정

우선 간단한 서비스를 만들어준다.

```
server{
	listen 8888;
	server_name localhost;

	location /post {
		alias /home/smpringles;
		index main.html;
	}
}
```
localhost의 8888번 포트 '/post' 도메인으로 접속을 하면 -> 서버측 pc의 /home/smpringles/main.html 파일을 보여준다.



그 후, 다른 포트로 서비스를 만든 뒤, `proxy_pass`로 요청을 리다이렉션 해준다.

```
server{
	listen 9999;
	server_name localhost;

	location / {
		proxy_pass http://localhost:8888;
	}
}
```
