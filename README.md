# **Kong API Gateway**

<img src="https://github.com/hoseokjang/Kong-api-gateway/assets/72066274/52ecb525-744f-4528-b39c-80228e8ae9d2" width="800" height="600">

### 📌 특징
- Nginx + Cassandra + Lua Script 기반
    - HTTPS와 S-HTTP를 통해 자체 Admin API를 통해 리버스 프록시 지시문을 수락하고, lua Script 언어를 사용하여 nginx 구성으로 변환하여 마이크로서비스 백엔드를 위한 게이트웨이 플랫폼을 구축합니다.
- 밀리초 미만의 처리 대기 시간 지원 및 높은 처리량, 고성능
- 풍부한 플러그인
    - Lua script로 custom plugin제작이 가능하고 kong-pongo를 통한 unittest, integration test 가능.
    - 플러그인을 통해 로드밸런싱 로깅, 인증, 속도제한, 변환 등을 제공.
- 클라우드 네이티브
    - 플랫폼에 구애받지 않으며 베어메탈에서 컨테이너에 이르기까지 모든 플랫폼에서 실행할 수 있고, 모든 클라우드에서 기본적으로 실행할 수 있습니다.
- 쿠버네티스 네이티브
    - Kong Ingress Controller를 사용하여 모든 L4+L7 트래픽을 라우팅하고 연결하는 네이티브 Kubernetes CRD로 kong을 선언적 리소스로 관리할 수 있습니다.
- 프록시, 고급 라우팅, 로드밸런싱, 서비스 모니터링 및 헬스체크, Service Discovery, Circuit-Breaker, 로깅, 속도 제한, 보안(ACL, 봇 감지, IP 허용/거부 등), 인증, 변환 (HTTP 요청 및 응답 조작), 캐싱, 클러스터링, 수평확장 등 다양한 기능을 제공하여 마이크로서비스 혹은 기존 API 트래픽을 쉽게 오케스트레이션 하기 위한 중앙 계층 역할을 합니다.
- 이러한 기능은 Restful API 혹은 선언적 구성을 통해 설정 가능합니다.

##### 단점
- 관리용 대시보드가 Enterprise 버전에만 제공됩니다.
    - CE 버전에서도 오픈소스 프로젝트인 Konga 와 Kong Dashborad 를 사용하면 보완 됨
- 설정 변경 등은 모두 RESTful API 요청으로 수행해야 합니다.
- 플러그인 개발이 어렵습니다.
    - Lua 언어를 따로 학습해야 합니다.


### 📌 아키텍처

<img src="https://github.com/hoseokjang/Kong-api-gateway/assets/72066274/3f2334f4-b708-426d-a08f-6c1b92a503c7" width="800" height="400">

### 📌 설치 과정

테스트 환경용으로 로컬 환경에 도커 이미지로 간단하게 설치

#### Kong 설치하기
- Kong API 게이트웨이는 DB가 필요하지만, 간편하게 DB가 없는 dbless 모드로 설치. 
- GUI를 이용하기 위해서 Konga (https://pantsel.github.io/konga/) 를 추가로 설치. 
    - Konga도 도커 컨테이너로 설치

##### 도커가 설치 되어 있는지 확인
```
$ sudo docker ps
$ sudo systemctl status docker
$ sudo systemctl start docker
$ sudo systemctl enable docker
```

##### 네트워크 생성하기
Kong API 게이트 웨이 Admin API를  Konga에서 호출할 수 있도록 같은 네트워크로 묶기 위해서 “kong-net”이라는 네트워크를 생성한다.
```
$ docker network create kong-net
``` 

##### 디스크 볼륨 생성하기
DB 없이 Kong API 게이트웨이를 설치하기 때문에, 디스크 볼륨을 정의하여 디스크에 설정 정보를 저장하도록 한다. 아래와 같이 디스크 볼륨을 생성한다.  
```
$ docker volume create kong-vol
```

##### 설정 파일 작성하기 
디폴트 설정 파일이 있어야 게이트웨이가 기동이 되기 때문에 디폴트 설정 파일을 만든다.

설정 파일은 앞서 만든 kong-vol 볼륨에 kong.yml 로 저장이 된다. 

파일을 생성하기 위해서 볼륨의 호스트 파일상의 경로를 알아보기 위해서 docker volume inspect 명령을 사용한다. 
 
```
$ docker volume inspect kong-vol
[
     {
         "CreatedAt": "2019-05-28T12:40:09Z",
         "Driver": "local",
         "Labels": {},
         "Mountpoint": "/var/lib/docker/volumes/kong-vol/_data",
         "Name": "kong-vol",
         "Options": {},
         "Scope": "local"
     }
 ]
```

디렉토리가 /var/lib/docker/volumes/kong-vol/_data임을 확인할 수 있다. 호스트(로컬 PC)의 이 디렉토리에서 아래와 같이 kong.yml 파일을 생성한다. 

```
_format_version: "1.1"

services:
- name: my-service
  url: https://example.com
  plugins:
  - name: key-auth
  routes:
  - name: my-route
    paths:
    - /

consumers:
- username: my-user
  keyauth_credentials:
  - key: my-key
```

생성이 끝났으면 아래 도커 명령어를 이용해서 Kong API 게이트웨이를 기동
```
% docker run -d --name kong \
     --network=kong-net \
     -v "kong-vol:/usr/local/kong/declarative" \
     -e "KONG_DATABASE=off" \
     -e "KONG_DECLARATIVE_CONFIG=/usr/local/kong/declarative/kong.yml" \
     -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
     -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
     -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
     -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
     -p 8000:8000 \
     -p 8443:8443 \
     -p 8001:8001 \
     -p 8444:8444 \
     kong:latest
```

설치가 제대로 되었는지
```
%curl http://localhost:8001 
```
명령을 이용하여 확인한다. 

*****************************************************************************************************************************************

#### Konga 설치 하기
다음은 웹 UI를 위해서 Konga를 설치한다. https://pantsel.github.io/konga/

여러가지 설치 방법이 있지만 간단하게 하기 위해서 역시 도커로 설치한다. 
```
% docker run -d --name konga \
	--network=kong-net \
	-v /var/data/kongadata:/app/kongadata \
	-p 1337:1337 \
	-e "NODE_ENV=production" pantsel/konga
```
 

이때, Kong API 서버 8001 포트로 통신을 해서 관리 API를 호출해야 하기 때문에, network를 "kong-net”으로 설정해서 실행한다. 

설치가 끝나면 http://mgt_kong.co.kr을 접속(윈도우  hosts 에 등록必)하면 Konga UI가 나오고, 관리자 계정과 비밀 번호를 입력할 수 있다.
 
로그인을 하면 Kong API 게이트웨이와 연결을 해야 하는데, 이름(임의로 적어도 된다.) Kong API 게이트웨이 Admin URL을 적어야 한다. 

<img src="https://github.com/hoseokjang/Kong-api-gateway/assets/72066274/2ec8429e-d9e1-43e6-bd01-c91ed7249f51" width="800" height="600">

이때 http://kong:8001 을 적는다. (kong은 Kong API 게이트 웨이를 도커 컨테이너로 기동할때 사용한 컨테이너 이름이다.)
