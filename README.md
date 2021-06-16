# Docker-Tutorials

> [Docker Get Started](https://docs.docker.com/get-started)를 참조 하여 작성함.  
> powershell 기반 명령어를 사용함.

## 컨테이너 이미지 빌드

아래 내용으로 Dockerfile을 생성한다.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:12-alpine
RUN apk add --no-cache python g++ make
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
```

`# syntax=docker/dockerfile:1`  
Parser Directives(파서 지시문)의 syntax diriective(구문 지시문) 이다. Dockerfile 빌드하는데 사용할 Dockerfile syntax 위치와 버전을 정의한다. [BuildKit](https://docs.docker.com/engine/reference/builder/#buildkit) 백엔드를 사용할 때만 사용할 수 있으며 클래식 빌더 백엔드를 사용할 때는 무시된다. 그렇다면 BuildKit이란 무엇인가? 간단하게 정리 하면 표준 Docker 명령어를 업그레이드 한 버전이라 볼 수 있다. 기존 명령어에 비해서 상당한 체감을 줄만한 요소들이 많으니 반드시 활성화 해서 사용하는게 좋을듯하다. ([활성화 방법]( https://brianchristner.io/what-is-docker-buildkit)) BuildKit에 대해서는 추가적으로 공부를 더하고 정리할 예정이다.

다시 돌아와서 해당 구문을 사용하는 이유는 무엇일까? 공식 문서를 참조 하면 다음과 같다.

* Automatically get bugfixes without updating the Docker daemon
* Make sure all users are using the same implementation to build your Dockerfile
* Use the latest features without updating the Docker daemon
* Try out new features or third-party features before they are integrated in the Docker daemon
* Use alternative build definitions, or create your own

더 줄여서 얘기 하면 syntax를 사용하면 Docker를 업그레이드하지 않고 선언한 버전으로 docker build를 실행한다는 말이다. 신기방기한 세상이다.

`FROM node:12-alpine`  
Docker Hub에서 가져올 base image이다.

`RUN apk add --no-cache python g++ make`  
해당 이미지에 python, g++, make 패키지를 설치한다.

`WORKDIR /app`  
WORKDIR은 RUN, CMD, ENTRYPOINT, COPY 및 ADD의 명령이 실행될 directory를 설정하는 명령어 이다.

`COPY . .`  
COPY 명령은 새 파일 또는 디렉토리를 복사하여 경로에있는 컨테이너의 파일 시스템에 추가하는 명령어 이다. 해당 명령은 dockerfile이 있는 위치의 모든 파일 및 폴더를 컨테이너 파일 시스템 경로에 복사한다는 내용이다.

`RUN yarn install --production`  
yarn명령어를 사용하여 해당 앱을 packaging 한다.

`CMD ["node", "src/index.js"]`  
CMD 명령어의 주요 목적은 실행중인 container에 기본값 제공이다. 딱 보면 실행인데 실행이 아니라 기본값 제공이라서 좀 당황스러울수 있는데 image build 완료후 이미지에 기본값을 전달하는데 그 기본값에는 변수 뿐만 아니라 실행 파일을 포함할 수도 있다라고 생각하면된다.  
RUN이랑 뭐가 다른가 의문을 가질수도 있는데 RUN은 image build 실행중 사용하는 실행이고 CMD는 빌드가 끝나고 난뒤에 실행을 의미한다.  

Dockerfile 생성했다면 이미지를 빌드해보자.

```powershell
docker build -t actionmandocker/docker-tutorial .
```

`-t`  
해당 이미지의 tag를 지정하기 위한 옵션이다.

`.`  
context를 지정한 것으로 docker build process는 context내 모든 파일들을 접근할 수 있다. __Dockerfile__ 의 경우 context내에 위치 시킨다.

이미지가 생성되었다면 컨테이너를 시작해보자.

```powershell
docker run -dp 3000:3000 actionmandocker/docker-tutorial
```

## 컨테이너 볼륨

컨테이너 volume 유형에는 named volume과 bind mount가 있다. 먼저 named volume을 알아보자.  

### Named volume

아래와 같이 volume을 생성해보자.

```powershell
docker volume create todo-db
```

생성한 volume과 컨테이너를 연결하여 시작해보자.

```powershell
docker run -dp 3000:3000 -v todo-db:/etc/todos actionmandocker/docker-tutorial
```

### Bind mount

아래와 같이 컨테이너를 시작해보자.  

```powershell
docker run -dp 3000:3000 `
    -w /app -v "$(pwd):/app" `
    node:12-alpine `
    sh -c "yarn install && yarn run dev"
```

로컬에서 Dockerfile로 생성한 이미지가 아니라 docker hub에 위치한 이미지를 사용하여 컨테이너를 작동시켰다.  

`-w`  
명령할 directory를 설정하는 옵션이다.

`pwd`  
docker 명령어는 아니고 bash나 powershell에서 현재 경로를 조회하는 명령어 이다.  

__-v__ 옵션을 보면 Bind mount의 경우 host의 mountpoint를 정확하게 지정할 수 있다. 이러한 특징 때문에 소스코드를 직접 mount시켜서 변경된 내용이 container에 즉시 반영되게 할 수 있다. 참고로 named volume의 경우 docker에서 임의 지정 경로에 mountpoint를 지정한다.

## 다중 컨테이너

그동안 작업했던 to-do app 컨테이너에 mysql 컨테이너를 추가하려고 한다.  

Container끼리 통신을 위해서는 한 네트워크 안에 있어야 한다.  
아래와 같이 네트워크를 생성해보자.

```powershell
docker network create todo-app
```

아래와 같이 mysql 컨테이너를 생성해보자.

```powershall
docker run -d `
    --network todo-app --network-alias mysql `
    -v todo-mysql-data:/var/lib/mysql `
    -e MYSQL_ROOT_PASSWORD=secret `
    -e MYSQL_DATABASE=todos `
    mysql:5.7
```

`--network`  
사용할 네트워크 옵션으로 위에 만들었던 todo-app network를 사용한다.

`-e`  
ENV(환경 변수) 옵션으로 container의 환경 변수로 전달된다.

이제 생성된 container 내부로 접속해보자.

```powershell
docker exec -it <mysql-container-id> mysql -u root -p
```

`-it`  
대화형, 가상 터미널 연결 옵션으로 -i, -t를 합쳐서 -it로 주로 사용한다.

`mysql -u root -p`  
컨테이너에서 실행할 명령어로 변수이다. 참고로 __-u__ 옵션은 user __-p__ 옵션은 패스워드를 입력하기 위한 옵션, 마지막으로 -p뒤에 사용할 데이터베이스명을 입력한다.

이제 todo app 컨테이너와 mysql 컨테이너를 연결해 보자.

```powershell
docker run -dp 3000:3000 `
    -w /app -v "$(pwd):/app" `
    --network todo-app `
    -e MYSQL_HOST=mysql `
    -e MYSQL_USER=root `
    -e MYSQL_PASSWORD=secret `
    -e MYSQL_DB=todos `
    node:12-alpine `
    sh -c "yarn install && yarn run dev"
```

컨테이너가 올라왔으면 브라우저에서 할 일을 입력하고 아래 명령어를 실행하여 컨테이너와 접속해보자.

```powershell
docker exec -it <mysql-container-id> mysql -p todos
```

mysql에 접속하였으면 테이블을 조회해보자.

```sql
select * from todo_items;
```

## 레이어 캐싱

Docker 이미지는 여러겹의 layer로 구성된다. 왜 그렇게 만들었을까? 한번 생성된 layer는 docker내부에 caching 된다. 동일한 layer 생성시 새로 생성하는 것이 아니라 caching된 layer를 사용하여 빌드 속도를 빠르게 하려는 목적이다.  
그렇다면 어떻게 해야 layer를 최대한 caching되도록 할 수 있을까? Docker는 상위 layer가 변경되어 caching된 이미지를 사용할 수 없을시 하위 layer도 새로 layer를 생성한다. 자주 변경될 수 있는 layer를 밑에 놔둬야 caching layer를 많이 사용한다는 얘기다.

아래는 기존 Dockerfile 및 2번째 이미지 생성시 logging이다. (1번째의 경우 caching된 데이터가 없기때문에 2번째 loggin을 사용한다.)

```dockerfile
# syntax=docker/dockerfile:1
FROM node:12-alpine
RUN apk add --no-cache python g++ make
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
```

```powershell
 => [internal] load build definition from Dockerfile              0.0s 
 => => transferring dockerfile: 32B     0.0s 
 => [internal] load .dockerignore       0.0s 
 => => transferring context: 2B         0.0s 
 => resolve image config for docker.io/docker/dockerfile:1        2.0s 
 => CACHED docker-image://docker.io/docker/dockerfile:1@sha256:e2a8561e419ab1ba6b2fe6cbdf49fd92b95912df1cf7d313c3e2230a333fdbcc  0.0s 
 => [internal] load metadata for docker.io/library/node:12-alpine 0.0s 
 => [1/5] FROM docker.io/library/node:12-alpine                   0.0s 
 => [internal] load build context       2.3s 
 => => transferring context: 551.66kB   2.3s 
 => CACHED [2/5] RUN apk add --no-cache python g++ make           0.0s 
 => CACHED [3/5] WORKDIR /app           0.0s 
 => CACHED [4/5] COPY . .               0.0s 
 => [5/5] RUN yarn install --production27.0s 
 => exporting to image                  2.6s 
 => => exporting layers                 2.5s 
 => => writing image sha256:78d62bed40a9a32a3956488084f241e3008a8be80e7dfed9d724222d376e96da           0.0s 
 => => naming to docker.io/actionmandocker/docker-tutorial        0.0s 
```

`RUN yarn install --production` layer에서 caching되지 않아 27초가 걸린다. Node-base 기반의 어플리케이션의 경우 install시 종속성을 다운받는데 종속성 데이터가 수정되는 일은 드물기 때문에 해당부분을 caching하여 빌드 속도를 높여보자.

먼저 Dockerfile을 수정하였다.

```dockerfile
# syntax=docker/dockerfile:1
FROM node:12-alpine
WORKDIR /app
COPY package.json yarn.lock ./
RUN yarn install --production
COPY . .
CMD ["node", "src/index.js"]
```

종속성 정보가 담긴 파일을 먼저 복사하여 종속성 데이터를 다운로드 받고 소스 데이터만 나중에 복사하도록 로직을 수정하였다. `COPY . .` 에서 Node 종속성 데이터가 들어 있는 `node_modules` 폴더를 복사하지 않도록 `.dokerignore` 파일을 작성했다.

___.dokerignore___:

```dockerfile
node_modules
```

2번째 이미지 생성시 logging이다.

```powershell
PS C:\csapi\workspace\docker-tutorials> docker build -t actionmandocker/docker-tutorial .
[+] Building 4.1s (13/13) FINISHED
 => [internal] load build definition from Dockerfile0.0s 
 => => transferring dockerfile: 32B                 0.0s 
 => [internal] load .dockerignore                   0.0s 
 => => transferring context: 34B                    0.0s 
 => resolve image config for docker.io/docker/dockerfile:1                    3.4s 
 => [auth] docker/dockerfile:pull token for registry-1.docker.io              0.0s 
 => CACHED docker-image://docker.io/docker/dockerfile:1@sha256:e2a8561e419ab1ba6b2fe6cbdf49fd92b95912df1cf7d313c3e2230a333fdbcc             0.0s 
 => [internal] load metadata for docker.io/library/node:12-alpine             0.0s 
 => [1/5] FROM docker.io/library/node:12-alpine     0.0s 
 => [internal] load build context                   0.1s 
 => => transferring context: 15.59kB                0.0s 
 => CACHED [2/5] WORKDIR /app                       0.0s 
 => CACHED [3/5] COPY package.json yarn.lock ./     0.0s 
 => CACHED [4/5] RUN yarn install --production      0.0s 
 => [5/5] COPY . .        0.1s 
 => exporting to image    0.1s 
 => => exporting layers   0.1s 
 => => writing image sha256:e5a485cdf98bddb9b51c2d8b5c68a443d3e98e0dbc05f0ec6ac12943d2cfa435            0.0s 
 => => naming to docker.io/actionmandocker/docker-tutorial                    0.0s 
```

## 다단계 빌드

Docker는 다단계 빌드를 제공한다. 다단계 빌드의 장점은 다음과 같다.

* 런타임 종속성과 빌드 타임 종속성의 분리
* 앱 구동시 꼭 필요한 사항만 담기 때문에 이미지 크기를 줄임

```dockerfile
# syntax=docker/dockerfile:1
FROM node:12 AS build
WORKDIR /app
COPY package* yarn.lock ./
RUN yarn install
COPY public ./public
COPY src ./src
RUN yarn run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```
