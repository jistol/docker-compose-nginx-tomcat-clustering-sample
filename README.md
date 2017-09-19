docker-compose를 이용한 Nginx + Tomcat 클러스터링 샘플
---
docker-compose를 이용하여 클러스터링 테스트 환경을 구축하는 샘플입니다.    

기본구조
----
2대의 Tomcat 컨테이너를 올리고 앞단에 Nginx로 reverse proxy 합니다.   
2대의 Tomcat은 가장 기본적인 클러스터링 설정을 사용하며 multicast 방식에 의해 세션 공유를 합니다.    

## 서버 구성도 ##    
![server structure](https://raw.githubusercontent.com/jistol/docker-compose-nginx-tomcat-clustering-sample/master/img/1.png)     

## 샘플 폴더 구조 ##    
![sample file list](https://raw.githubusercontent.com/jistol/docker-compose-nginx-tomcat-clustering-sample/master/img/2.png)     
    
Nginx와 Tomcat의 설정은 각 폴더별로 구분하고 Docker Build시 copy하도록 설정해두었습니다.     

docker-compose.yml 설정
----
docker명령어로 일일히 다 올리기 귀찮기 때문에 docker-compose를 이용하여 한방에 올립니다.   
docker-compose에 관한 자세한 사항은 [docker compose doc](https://docs.docker.com/compose/)을 참고하세요.    

```yaml
# 일단 버전은 3을 사용합니다. 덕분에 extends기능이 없어졌더군요 ;(
version: '3'

# 각 서비스 컨테이너를 정의합니다. ( nginx * 1 + tomcat * 2 )
services:
    # tomcat 1번 서버입니다. 같은 설정을 2번에서도 사용하기 때문에 &was로 명명하고 tomcat2에서 참조합니다.
    # tomcat1 서비스의 모든 관련파일은 ./tomcat1 폴더에서 가져옵니다. 
    tomcat1: &was
        # tomcat 기동시 java option값을 추가하기 위해 아래 설정을 추가했습니다. 
        environment:
            - JAVA_OPTS=-Dspring.profiles.active=docker -Dfile.encoding=euc-kr
        build: 
            context: .
            # Dockerfile을 실행시 conf/server.xml과 webapps에 파일 배포를 위해 argument를 추가합니다.
            # 아래 값은 DockerfileTomcat에서 사용됩니다.
            args:
                conf: tomcat1/conf
                warpath: tomcat1/webapps/ROOT.war
            # tomcat 서버 이미지 빌드를 위한 Dockerfile을 별도로 지정해줍니다.
            dockerfile: ./DockerfileTomcat
        # 
        # Docker 컨테이너에 붙지 않고 각 tomcat서버의 로그를 따로 확인하기 위해 외부 저장소와 연결합니다.
        volumes:
            - ./tomcat1/logs/:/usr/local/tomcat/logs/
    
    # tomcat2번 서버입니다. tomcat1 서비스에서 설정한 내용을 그대로 사용하고 달라지는 설정에 대해서는 아래와 같이 직접 입력해줍니다.
    tomcat2:
        <<: *was
        build: 
            context: .
            args:
                conf: tomcat2/conf
                warpath: tomcat2/webapps/ROOT.war
            dockerfile: ./DockerfileTomcat
        volumes:
            - ./tomcat2/logs/:/usr/local/tomcat/logs/
    # nginx 설정입니다.
    nginx:
        build:
            context: .
            # nginx 서버 이미지 빌드를 위한 Dockerfile을 별도로 지정해줍니다.
            dockerfile: ./DockerfileNginx
            # Dockerfile실행시 conf/nginx.conf파일이 복사 될 수 있도록 argument를 추가합니다.
            # 아래 값은 DockerfileNginx에서 사용됩니다.
            args:
                conf: nginx/conf
        # 외부에서 직접 8080포트로 붙어야 하기 때문에 컨테이너 포트를 외부로 열어줍니다.
        ports: 
            - "8080:8080"
```

위에서 설정한 설정에서 참조값들을 모두 적용한 문서를 보고 싶을 때는 아래와 같은 명령어로 실행할 수 있습니다.

```console
$ docker-compose config
```

Dockerfile 설정
----
위에서 docker-compose 실행시 각 컨테이너가 Dockerfile을 실행하도록 설정하였습니다.    
다음과 같이 nginx / tomcat 용 Dockerfile을 생성합니다.

## DockerfileNginx ##
```Dockerfile
FROM nginx:latest
MAINTAINER jistol <pptwenty@gmail.com>

ARG conf

COPY $conf/nginx.conf /etc/nginx/nginx.conf

WORKDIR /usr/local/tomcat/bin
CMD ["nginx", "-g", "daemon off;"]
```

`ARG conf`값은 docker-compose.yml에 설정되어 있습니다.    

## DockerfileTomcat ##
```Dockerfile
FROM tomcat:latest
MAINTAINER jistol <pptwenty@gmail.com>

ARG conf
ARG warpath

RUN rm -rf /usr/local/tomcat/webapps/*
COPY $conf/* /usr/local/tomcat/conf/
COPY $warpath /usr/local/tomcat/webapps/ROOT.war

WORKDIR /usr/local/tomcat/bin
CMD ["catalina.sh", "run"]
```

`ARG conf`, `ARG warpath`값은 docker-compose.yml에 설정되어 있습니다.    


Tomcat 설정
----
`server.xml`파일에 아래와 같이 설정합니다.

```xml
<!-- server.xml -->
<Cluster className="org.apache.catalina.ha.tcp.SimpleTcpCluster"/>
```

자세한 설정은 [Tomcat 8 세션 클러스터링 하기](/java/2017/09/15/tomcat-clustering/)에 포스팅한 내용을 참고하세요.     

Nginx 설정
----
`nginx.conf`파일에 reverse proxy를 위한 설정을 합니다.

```conf
http {
    
    ...
    
    # proxy 설정할 서버목록을 만듭니다.
    # host명은 docker 컨테이너의 service 이름과 동일하게 맞추어 줍니다. 
    upstream was-list {
        server tomcat1:8080;
        server tomcat2:8080;
    }
    
    ...
    
    server {
    
        # nginx 서버가 8080을 listen하도록 설정합니다.
        listen       8080;
        server_name  localhost;

        # 8080포트로 들어오는 모든 요청을 위에 설정한 'was-list'그룹으로 보냅니다.
        location / {
            proxy_pass http://was-list;
        }
    }
    
    ...
    
}
```

실행
----
다음과 같이 실행합니다.

```console
$ docker-compose up -d
```

`-d` 옵션을 사용해야 실행후 각 컨테이너 콘솔 화면에서 detach됩니다.   

기본적으로 docker-compose 실행시 이미지를 기존 빌드된 것으로 캐쉬하기 때문에 설정파일이나 배포파일이 바뀔 경우 아래와 같이 실행하여 새로 이미지를 만들도록 합니다.    

```console
$ docker-compose up -d --build
``` 

실행시 아래와 같이 이미지 빌드 로그와 함께 각 컨테이너가 실행되는 것을 볼 수 있습니다.


```console
$ docker-compose up -d --build
Building nginx
Step 1/6 : FROM nginx:latest
 ---> da5939581ac8
Step 2/6 : MAINTAINER jistol <pptwenty@gmail.com>
 ---> Using cache
 ---> 2d9af4961368
Step 3/6 : ARG conf
 ---> Using cache
 ---> ead97bc0569d
Step 4/6 : COPY $conf/nginx.conf /etc/nginx/nginx.conf
 ---> Using cache
 ---> 5ab748ec2a17
Step 5/6 : WORKDIR /usr/local/tomcat/bin
 ---> Using cache
 ---> 3eabdd2a3dd5
Step 6/6 : CMD nginx -g daemon off;
 ---> Using cache
 ---> 7f4f2405e032
Successfully built 7f4f2405e032
Successfully tagged tomcatdocker1_nginx:latest
Building tomcat2
Step 1/9 : FROM tomcat:latest
 ---> 0fbedce2f08c
Step 2/9 : MAINTAINER jistol <pptwenty@gmail.com>
 ---> Using cache
 ---> 13adee263b5d
Step 3/9 : ARG conf
 ---> Using cache
 ---> 57a78bc9e8ce
Step 4/9 : ARG warpath
 ---> Using cache
 ---> 638fca357d24
Step 5/9 : RUN rm -rf /usr/local/tomcat/webapps/*
 ---> Using cache
 ---> 928ef1b94bb2
Step 6/9 : COPY $conf/* /usr/local/tomcat/conf/
 ---> Using cache
 ---> 30f8faae0012
Step 7/9 : COPY $warpath /usr/local/tomcat/webapps/ROOT.war
 ---> 3bd3ddeff2d6
Removing intermediate container 9c0b330ce6f6
Step 8/9 : WORKDIR /usr/local/tomcat/bin
 ---> dc773d206d87
Removing intermediate container 693e8b125384
Step 9/9 : CMD catalina.sh run
 ---> Running in c430115e5460
 ---> 3879504509c0
Removing intermediate container c430115e5460
Successfully built 3879504509c0
Successfully tagged tomcatdocker1_tomcat2:latest
Building tomcat1
Step 1/9 : FROM tomcat:latest
 ---> 0fbedce2f08c
Step 2/9 : MAINTAINER jistol <pptwenty@gmail.com>
 ---> Using cache
 ---> 13adee263b5d
Step 3/9 : ARG conf
 ---> Using cache
 ---> 57a78bc9e8ce
Step 4/9 : ARG warpath
 ---> Using cache
 ---> 638fca357d24
Step 5/9 : RUN rm -rf /usr/local/tomcat/webapps/*
 ---> Using cache
 ---> 26dea2133cee
Step 6/9 : COPY $conf/* /usr/local/tomcat/conf/
 ---> Using cache
 ---> c546b169bf25
Step 7/9 : COPY $warpath /usr/local/tomcat/webapps/ROOT.war
 ---> 4f1edfc42b8d
Removing intermediate container f02a70122c3a
Step 8/9 : WORKDIR /usr/local/tomcat/bin
 ---> 50683e072451
Removing intermediate container e2ca57d27775
Step 9/9 : CMD catalina.sh run
 ---> Running in 8da01a9227c2
 ---> 1e050e465174
Removing intermediate container 8da01a9227c2
Successfully built 1e050e465174
Successfully tagged tomcatdocker1_tomcat1:latest
Creating tomcatdocker1_nginx_1 ... 
Creating tomcatdocker1_tomcat1_1 ... 
Creating tomcatdocker1_tomcat2_1 ... 
Creating tomcatdocker1_nginx_1
Creating tomcatdocker1_tomcat1_1
Creating tomcatdocker1_tomcat1_1 ... done
```

docker 프로세스를 확인해보면 다음과 같이 3개의 컨테이너가 올라간 것을 확인 할 수 있습니다.

```console
$ docker ps -a
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS                     PORTS                                                NAMES
1e2240de4a77        tomcatdocker1_tomcat1    "catalina.sh run"        4 minutes ago       Up 4 minutes               8080/tcp                                             tomcatdocker1_tomcat1_1
9706f7d0f85a        tomcatdocker1_tomcat2    "catalina.sh run"        4 minutes ago       Up 4 minutes               8080/tcp                                             tomcatdocker1_tomcat2_1
20aa7abec733        tomcatdocker1_nginx      "nginx -g 'daemon ..."   4 minutes ago       Up 4 minutes               80/tcp, 0.0.0.0:8080->8080/tcp                       tomcatdocker1_nginx_1
```

중지
----
아래 명령어를 통해 중지할 수 있습니다.

```console
$ docker-compose down
```