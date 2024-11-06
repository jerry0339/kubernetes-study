# Docker 명령어
* 공식 문서 - https://docs.docker.com/reference/cli/docker/compose/

```bash
docker login # docker hub 로그인
docker search {이미지명} # 도커 허브에서 이미지 검색
docker images # 이미지 리스트 확인
docker image ls # images랑 같음

# 이미지 빌드 
docker build -t {이미지명}:{태그} "." # Dockerfile이 현재 디렉토리에 위치한 경우 ("."은 현재 디렉토리를 의미함)
docker build -f {Dockerfile} -t {이미지명}:{태그} # -f 옵션으로 도커파일 경로 지정 가능

# 이미지 Docker Hub에 푸시
docker push {도커허브Id}/{이미지명}:{태그명} # `{허브id}/`가 반드시 포함되어야 함

# 이미지 Docker Hub로부터 pull
docker pull jerry0339/discovery:latest

# 이미지 복사 & 이미지 이름/태그 변경
docker image tag {기존의 이미지명} {새로운 이미지명} # tag는 기존의 이미지의 tag로 적용
docker image tag {기존의 이미지명} {새로운 이미지명}:{새로운 태그명} #
docker image tag {기존의 이미지명}:{기존의 태그명} {새로운 이미지명}:{새로운 태그명}

# 이미지 정보 출력
docker image inspect {이미지명}
docker image inspect --format='{{.Config.Labels}}' {이미지명} # LABEL 항목만 필터링해서 출력

# 이미지 삭제
docker rmi {이미지ID} # 이미지ID unique한 경우
docker rmi {이미지명} # 이미지명 unique한 경우
docker rmi {이미지명}:{태그} # 이미지 {이름:태그}로 지정 삭제

# --Todo--
# 컨테이너 관련 명령어
docker ps # 동작중인 컨테이너 확인
docker ps -a # 정지된 컨테이너 확인
docker rm {컨테이너ID} # 컨테이너 삭제
docker rm {컨테이너ID-1}, {컨테이너ID-2} # 컨테이너 복수개 삭제
docker rm `docker ps -a -q` # 컨테이너 모두 삭제
docker container inspect {컨테이너ID} # 컨테이너 정보 확인

# docker compose 관련 명령어
docker-compose push {서비스명} # 서비스의 이미지를 허브로 push
docker-compose pull {서비스명} # 서비스의 이미지를 허브에서 pull
docker-compose up # docker-compose.yml 파일에 정의돈 모든 서비스에 필요한 컨테이너를 생성하고 시작
docker-compose up -d # 백그라운드에서 실행 (detached mode).
docker-compose up {서비스명} # 특정 서비스만 실행
docker-compose down # 실행 중인 모든 서비스를 종료/제거하고 network나 volume또한 제거, c.f.잠시 중단은 stop
docker-compose down {서비스명} # 특정 서비스만 종료/제거
docker-compose ps # 현재 실행 중인 모든 서비스 컨테이너들의 상태를 확인
docker-compose ps {서비스명} # 특정 서비스 상태 확인
docker-compose logs -f -t # 서비스의 로그를 확인 (f:로그 출력을 따라감, t:타임 스탬프 표시)
docker-compose logs {서비스명} # 특정 서비스의 로그만 확인
docker-compose build # docker-compose.yml 파일에 정의된 서비스 이미지를 빌드
docker-compose build {서비스명} # 특정 서비스 이미지만 빌드
docker-compose stop # 모든 서비스 컨테이너 중단, docker-compose start로 시작 가능
docker-compose stop {서비스명} # 특정 서비스만 중단
docker-compose start {서비스명} # 중지된 서비스를 다시 시작
docker-compose restart {서비스명} # 실행 중인 서비스를 재시작 (stop + start), 이미지 변경시 build후 restart
docker-compose rm # 중지된 컨테이너들을 모두 삭제
docker-compose rm {서비스명} # 특정 중지된 컨테이너 삭제
docker-compose exec {서비스명} {command} # 실행 중인 서비스 내에서 명령어를 실행 (docker exec와 유사)

# exec 예시
# web이라는 서비스에 bash 쉘 실행 (i옵션:터미널을 통한 입력이 가능하도록, t옵션:컨테이너의 출력을 터미널로 전달)
docker-compose exec -it web /bin/bash
```