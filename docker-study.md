# Docker 로컬 환경 테스트

이미지를 인터넷이 되는 환경에서 만들고, 실제 사용은 인터넷이 불가능한 환경에서 로컬 서비스를 제공하기 위해서 방안을 찾던 중   
서비스를 Docker를 이용해서 로컬 이미지로 띄우고 싶을 경우 사용할 수 있는 방법과 그 방법간에 발견한 부분을 기록한 내용입니다.  

> 준비물
- Docker, Docker cli
- 배포할 파일(WAR, JAR, _static, ...)
- 컨테이너별 설정 파일 혹은 컨테이너 실행 후 실행될 shell 파일

## 1. 로컬 이미지 배포
### 1.1. 도커 이미지 저장
로컬로 이미지를 저장하는 방법, tar로 저장됨
```sh
#docker save -o 이미지이름.tar 이미지이름:태그
docker save -o nginx.tar test/nginx:1.0
```

### 1.2. tar 이미지를 불러오는 방법
`save` 명령어로 뽑아낸 tar를 도커에서 이미지로 불러오는 방법
```sh
#docker load -i 이미지이름.tar
docker load -i nginx.tar

# 이후 이미지 확인
docker images
```

### 1.3. 도커 Volume 생성
#### 1.3.1. Host 볼륨
실제 본인의 컴퓨터에 도커 컨테이너와 공유할 디렉토리 설정
```sh
docker run -v C:/volume/nginx/:/test-front -t test/nginx:0.1 .
```

#### 1.3.2. Docker 볼륨
Docker가 컨테이너와 공유하는 볼륨을 만들어 파일 등을 공유할 수 있도록 함(Docker가 종료되면 사용 불가)
```sh
# 볼륨 생성
docker volume create my_volume
# 확인
docker volume ls

# Run 연결
docker run -d -v my_volume:/path/in/container my_image
```

> Local Docker 설치
```sh
# 1. docker 설치 tar 풀기
# 도커 사이트에서 url 을 wget 으로 받아도 무관
tar -xvf 도커_설치_파일.tar 
cd 도커_설치_디렉토리
sudo cp -r * /usr/

# 2. 도커 서비스 시작
sudo systemctl start docker

# 3. 재부팅 옵션 설정(optional)
sudo systemctl enable docker
```

> Docker 라이선스  
**Docker Desktop**(UI 제공)은 비상업적 목적으로만 무료로 사용가능, **Docker CLI**(커맨드라인)은 무료 및 오픈소스이다. Docker Engine은 Apache 2.0 라이선스에 따라 무료 및 오픈소스로 제공됨  
 
# 2. 컨테이너별 배포 방법
## 2.1. nginx
nginx 이미지를 받아서 nginx가 WEBROOT로 사용할 폴더를 미리 volume으로 열어주고 실행 시 연결
> Dockerfile (nginx)
```Dockerfile
# Nginx를 사용하여 최종 이미지를 빌드합니다.
FROM nginx:alpine
# VOLUME 설정 (없을경우 자동 생성)
VOLUME [ "/test-front" ]
```

> nginx.conf

컨테이너 /test-front 디렉토리를 WEBROOT로 사용
```conf
server {
  listen 80;
  server_name localhost;

  root /test-front;
```

#### Run
실행 시 `-v`옵션으로 볼륨을 연결한다.
```sh
docker run -d -p 8080:80 -v test-front:/test-front --name test-nginx test/nginx:0.1
```

### 2.2. Front 컨테이너
프론트엔드 서비스를 nginx로 배포하는데, 이때 컨테이너의 역할은 단순히 볼륨에 소스를 붙여넣는 역할로 설정
> Dockerfile (front)
```Dockerfile
FROM ubuntu:18.04
VOLUME [ "/test-front" ]
# 실제 호스트 파일(폴더) 복사
COPY ./_static /usr/src/app
CMD ["./front.sh"]
```
> front.sh  

해당 쉘을 실행해 공유하는 볼륨에 소스를 배포하고, 빈 컨테이너로 작동하도록 설정(tail)
```sh
#!/bin/bash
cp -r /usr/src/app/* /test-front
tail -f /dev/null
```

#### Run
실행 시 `-v`옵션으로 볼륨을 연결한다.
```sh
docker run -d -v test-front:/test-front --name test-front test/front:0.1
```

## 2.3. Tomcat (WAS)
nginx 와 유사, 설정파일을 따로 구성해 붙여넣거나 `echo` 등의 명령어로 사용
```Dockerfile
FROM tomcat:8.5.98-jdk8
VOLUME [ "/usr/local/tomcat/webapps" ]
# 설정 파일 복사
COPY ./server.xml /usr/local/tomcat/conf/server.xml
```

### Run
포트 설정 및 볼륨 연결
```sh
docker run -d -p 28080:8080 -v test-back:/usr/local/tomcat/webapps --name test-tomcat test/tomcat:0.1
```

## 2.4. Back (WAR, JAR, etc...)
백엔드 서비스 코드를 배포하는데 이때 백엔드를 어떤걸 사용하느냐에 따라 다양하게 바뀔 수 있으나 지금은 Tomcat에 WAR를 배포하는 방법으로 사용

```Dockerfile
FROM ubuntu:18.04
RUN mkdir -p /test-back
VOLUME [ "/test-back" ]
# war 배포
COPY ./build/libs/test-api.war /usr/src/test-api.war
# 컨테이너 실행 후 실행
CMD ["./back.sh"]
```

> back.sh

소스를 배포하고 빈 컨테이너로 돌아가게 하는 Shell 파일
```sh
#!/bin/bash
cp /usr/src/test-api.war /test-back/test-api.war
tail -f /dev/null
```

#### Run
볼륨 연결을 통해 배포
```sh
docker run -d -v test-back:/test-back --name test-api test/api:0.1
```


## 2.5. DB (mariadb, mysql, etc...)
DB는 필수일 가능성이 높으나 일단 여기서는 mariadb로 사용  
```Dockerfile
# mariadb 10.3 이미지 Pull
FROM mariadb:10.3
# VOLUME 설정
VOLUME [ "/var/lib/mysql" ]
# 서비스 포트 설정
EXPOSE 3306
# 환경 변수 설정
ENV MYSQL_ROOT_PASSWORD=root
ENV MYSQL_DATABASE=test
ENV MYSQL_USER=test
ENV MYSQL_PASSWORD=test
# MariaDB 설정을 추가 (파일로 적용해도 무관)
RUN echo "[mysqld]" >> /etc/mysql/my.cnf \
    && echo "character-set-server=utf8mb4" >> /etc/mysql/my.cnf \
    && echo "collation-server=utf8mb4_unicode_ci" >> /etc/mysql/my.cnf
# init sql 위치에 복사 (컨테이너 실행 시 적용될 쿼리)
ADD ./test_20231117.sql /docker-entrypoint-initdb.d/test_20231117.sql
```

#### Run
직접 호스트와 연결하는 볼륨이 있도록 설정(SQL 데이터는 실제로 호스트에 남기는게 좋을지도)
```sh
docker run -d -p 13306:3306 -v C:\volume\mariadb:/var/lib/mysql --name test-mariadb test/mariadb:0.1
```

## 3. 기타사항
### 3.1. 개선 가능한 부분
#### 3.1.1. Docker Compose 구성
각각의 `Dockerfile`로 구성이 가능하기 때문에 이 부분들의 설정을 잘 묶어서 Docker Compose를 활용해 Compose 한 번의 실행으로 모든 컨테이너를 올릴 수 있도록 구성 가능할 것이다.

#### 3.1.2. 로컬 Hub 구성
이미지를 Save - Load 하는 것 보다 이미지를 담을 수 있는 LocalHub를 구성해 이미지 tar를 들고 다니는것이 아닌, Local에서 Push/Pull 할 수 있는 구성을 만들어 보는것도 좋을 것 같다.

### 3.2 기타 공부사항
#### 3.4. Dockerfile에서 RUN, CMD, ENTRYPOINT 차이
빌드를 위한 Dockerfile 생성 시  
- RUN : 이미지를 빌드하는 순간에 실행되는 명령어
- CMD : 컨테이너 실행 시 수행되는 명령어, `docker run` 명령에 마지막에 오는 명령어에 덮어 씌워지기 때문에 한 번만 수행되는 것이 좋고 실행 명령어 사용과 달라지는 것을 주의해야 함
- ENTRYPOINT : 컨테이너 실행 시 수행되는 명령어, `docker run` 명령에 마지막에 오는 명령을 인자(argument)로 받을 수 있음, 명령어를 복잡하게 만들거나 미리 설정한 ENTRYPOINT 명령의 인자로 적절히 활용할 수 있으나 명령어가 잘못합쳐져 오류가 발행할 수 있으니 확인 필요

#### 3.5. Docker Container 간의 통신
컨테이너간 네트워크 이용시 아래와 같이 localhost 가 아닌 **host.docker.internal** 사용
```yml
jdbc:mariadb://host.docker.internal:13306/test?
```