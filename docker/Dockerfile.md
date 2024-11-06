# Dockerfile
* 파일 이름은 반드시 `Dokcerfile`이라고 저장해야 함
* 이미지 생성시 도커파일 경로를 따로 지정해 주는게 아니라면, Dockerfile는 프로젝트의 최상위 경로에 위치시켜야 함
* `FROM`, `RUN`, `CMD`키워드는 Dockerfile에서 가장 많이 사용
* `FROM` 키워드는 반드시 처음 명령어로 시작해야 함
  * `FROM`에는 베이스 이미지 정보를 입력
    * ex. FROM ubuntu:18.04
  * 도커 허브에서 pull가능한 이미지를 검색할 수 있다. - [도커 허브 이미지 검색(java)](https://hub.docker.com/_/openjdk)
  * 명령어로도 검색 가능하다. ex. `docker search nginx`
* `CMD` 키워드는 이미지를 컨테이너로 띄울 때 한 번만 실행
* `RUN` 키워드는 이미지를 빌드하는 과정에서 실행할 명령어를 지정

<br>

## Dockerfile 기본 문법
* `FROM`: 베이스 이미지 지정 명령
  * ex. FROM httpd:alpine
* `LABEL`: 버전 정보, 작성자 등과 같은 이미지에 대한 설명을 작성하기 위한 명령어
  * ex. LABEL version="1.0.0"
* `RUN`: 도커파일로부터 도커 이미지를 빌드하는 순간에 실행이 되는 명령어
  * ex. RUN pip install -r requirements.txt
* `CMD`: 컨테이너가 생성되고 최초로 실행할 때(docker run) 수행되는 명령어
  * ex. CMD ["./mvnw", "clean", "package"]
* `ENTRYPOINT`: 컨테이너가 생성되고 최초로 실행할 때(docker run) 수행되는 명령어
  * ex. ENTRYPOINT ["java", "-jar", "app.jar"]
* `EXPOSE`: docker 컨테이너 외부에 오픈할 포트 설정
  * ex. EXPOSE 8080
* `ENV`: docker 컨테이너 내부에서 사용할 환경 변수 지정
  * ex. ENV PATH /usr/bin:$PATH
* `WORKDIR`: 컨테이너 내에서 작업 디렉토리를 설정. shell의 cd 명령문처럼 컨테이너 상에서 작업 디렉토리로 전환을 위해 사용
  * ex. WORKDIR /app
* `COPY`: 호스트 컴퓨터에 있는 디렉터리나 파일을 Docker이미지 내부로 복사하기위해 사용
  * ex. COPY ~/model ./model
* `VOLUME`: 호스트와 컨테이너 간에 공유할 디렉토리를 지정
  * ex. VOLUME /app/data

<br>

## RUN / CMD / ENTRYPOINT
* `RUN`은 이미지 빌드시 실행되는 명령어
* `CMD` & `ENTRYPOINT` 두 명령어는 모두 컨테이너가 생성되고 최초로 실행할 때 수행되는 명령어임
* `CMD`와 `ENTRYPOINT` 차이점?
  * ENTRYPOINT는 이미지 실행할때 명령어가 항상 실행이 됨
  * CMD는 이미지 실행 스크립트에 명령어를 추가로 넣으면 CMD의 명령어가 수행되지 않음. 즉 명령어를 실행할 때 변경 가능함
  * 예시 - 아래의 1번 CMD명령어가 들어간 이미지를 2번 명령어로 실행시키면 "CMD test"가 아니라 "hello"가 출력이 됨
    1. CMD ["echo", "CMD test"]
    2. docker run {이미지명} echo hello

<br>

## Dockerfile 예시
* SpringBoot 3, java 17, maven

```Dockerfile
FROM amazoncorretto:17.0.7-alpine
LABEL maintainer="jerry0339@naver.com"
LABEL version="1.0.0"
LABEL description="spring clude eureka discovery image for Dockerfile test"
LABEL createdDate="24-10-23"
CMD ["./mvnw", "clean", "package"]
ARG JAR_FILE_PATH=target/*.jar
COPY ${JAR_FILE_PATH} app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

* SpringBoot 3, java 17, gradle

```Dockerfile
FROM amazoncorretto:17.0.7-alpine
CMD ["./gradlew", "clean", "build"]
ARG JAR_FILE_PATH=build/libs/*.jar
COPY ${JAR_FILE_PATH} app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```