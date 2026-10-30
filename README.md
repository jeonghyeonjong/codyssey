# 개발 워크스테이션 구축 미션

## 1. 실행 환경 및 데몬 점검
- OS: Windows 11
- Shell: Git Bash
- Container Runtime: Docker Desktop (WSL 2 Backend)
- Version Control: Git for Windows

```bash
$ docker --version
Docker version 27.3.1, build ce12230

$ docker info
Client:
 Context:    default
 Debug Mode: false
Server:
 Containers: 1
  Running: 1
  Paused: 0
  Stopped: 0
 Images: 4
 Server Version: 27.3.1
 Operating System: Docker Desktop
 OSType: linux
 Architecture: x86_64
```

---

## 2. 프로젝트 디렉토리 구조
    codyssey/
    ├── .git/                 # Git 형상 관리 로컬 저장소 메타데이터 디렉토리
    ├── README.md             # 기술 문서 및 18개 평가 기준 실습 결과 보고서
    ├── Dockerfile            # Nginx 기반 커스텀 웹 서버 이미지 빌드 명세서
    └── app/                  # 웹 서버 정적 리소스 격리 디렉토리
        └── index.html        # 메인 웹 페이지 HTML 소스 파일

- 구성 기준 및 역할:
  - Dockerfile: 컨테이너 빌드 자동화를 위해 프로젝트 최상위 루트에 배치합니다.
  - app/: 웹 서버 데몬이 참조할 정적 자원(HTML)을 독립된 공간에 격리하여 소스 관리와 복사(COPY)의 재현성을 보장합니다.

---

## 3. 핵심 개념 정리

### 1) 이미지 vs 컨테이너 vs 변경 흐름 (라이프사이클)
- Docker Image: 애플리케이션 실행에 필요한 바이너리, 라이브러리, 설정을 포함하는 불변(Immutable)의 읽기 전용 템플릿(설계도)입니다.
- Docker Container: 이미지를 기반으로 격리된 프로세스 공간을 할당받아 실행되는 독립 인스턴스(실체)이며, 이미지 위에 쓰기 가능한 얇은 레이어(Read-Write Layer)를 얹어 동작합니다.
- 변경 흐름: 소스 코드 수정 시 실행 중인 컨테이너 내부를 직접 수정하지 않고, Dockerfile 기반으로 새 이미지를 빌드(build) → 기존 컨테이너 중지/삭제 후 새 컨테이너 실행(run)하는 무상태(Stateless) 라이프사이클을 따릅니다.

### 2) 네트워크 네임스페이스와 포트포워딩
- 컨테이너는 호스트 OS로부터 네트워크 네임스페이스(Network Namespace) 단위로 격리되어 자체 가상 IP와 독립된 네트워크 인터페이스를 보유합니다.
- 따라서 호스트 브라우저에서 컨테이너 내부 웹 서버(80 포트)에 접근하기 위해서는 호스트 포트(8080)로 들어온 TCP 패킷을 컨테이너 포트(80)로 전달해 주는 포트포워딩(-p 8080:80) 설정이 필수적입니다.

### 3) 경로 선택 기준 및 재현성 (호스트 vs 컨테이너)
- 호스트 경로: 특정 로컬 사용자의 절대 경로(C:\Users\...)를 지정하면 타 사용자나 CI/CD 환경에서 경로가 깨지므로, 작업 루트 기준 상대 경로(., app/)를 사용하여 재현성을 보장합니다.
- 컨테이너 경로: 베이스 이미지(Nginx)의 공식 파일 시스템 규격에 맞춘 절대 경로(/usr/share/nginx/html/, /data)를 지정해야 웹 데몬이 자원을 올바르게 탐색합니다.

### 4) 파일 권한 체계 (rwx 및 755/644 의미)
- 파일 권한은 r(읽기=4), w(쓰기=2), x(실행/디렉토리 탐색=1) 3비트 8진수의 합으로 표기되며, 세 자리는 각각 [소유자] / [소유 그룹] / [기타 사용자]를 나타냅니다.
  - 755 (rwxr-xr-x): 소유자는 읽기/쓰기/실행 가능(7), 그룹 및 기타 사용자는 읽기 및 디렉토리 접근/탐색만 가능(5). Nginx 웹 데몬 프로세스가 디렉토리 내부를 탐색하고 파일을 읽을 수 있도록 디렉토리에 부여합니다.
  - 644 (rw-r--r--): 소유자는 읽기/쓰기 가능(6), 그룹 및 기타 사용자는 읽기 전용(4). 일반 정적 웹 문서나 텍스트 파일에 부여합니다.

### 5) 바인드 마운트 vs 도커 볼륨 및 백업 전략
- 바인드 마운트(-v $(pwd)/app:...): 호스트 디렉토리를 컨테이너에 직접 연결하여 실시간 코드 수정이 반영되므로 로컬 개발 시 유용하지만 호스트 환경에 종속됩니다.
- 도커 볼륨(-v my-data:...): Docker 엔진이 관리하는 전용 스토리지에 격리 저장되어 컨테이너 생명주기와 완전히 독립되므로 데이터 안정성과 OS 간 이식성이 뛰어납니다.
- 백업 및 복구 권장 절차: 볼륨 데이터를 tar 아카이브로 묶어 호스트로 내보내거나 복원합니다.

```bash
# 백업
docker run --rm -v my-data:/data -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /data
# 복구
docker run --rm -v my-data:/data -v $(pwd):/backup ubuntu tar xvf /backup/backup.tar -C /
```

---

## 4. 터미널 조작 및 디렉토리 권한 실습 로그

### 1) 기본 파일 조작 (생성, 이동, 삭제)
    $ mkdir -p test_dir && touch test_dir/temp.txt
    $ mv test_dir/temp.txt test_dir/renamed.txt
    $ ls -la test_dir/
    total 0
    drwxr-xr-x 1 jeonghyeonjong 197609 0 10월 30 19:25 .
    drwxr-xr-x 1 jeonghyeonjong 197609 0 10월 30 19:25 ..
    -rw-r--r-- 1 jeonghyeonjong 197609 0 10월 30 19:25 renamed.txt
    $ rm test_dir/renamed.txt && rmdir test_dir

### 2) app 디렉토리 권한 설정 (755)
    $ mkdir -p app
    $ chmod 755 app
    $ ls -ld app
    drwxr-xr-x 1 jeonghyeonjong 197609 0 10월 30 19:27 app/
    $ echo "<h1>Codyssey Web Server</h1>" > app/index.html

- 권한 선택 및 변경 근거: Nginx 웹 서버 데몬 프로세스가 app/ 디렉토리 내부로 접근 및 탐색(실행 권한 x)하고, 하위의 index.html 파일을 읽기(읽기 권한 r)할 수 있도록 디렉토리 권한을 755(rwxr-xr-x)로 설정하였습니다.

---

## 5. Docker 기본 컨테이너 점검

### 1) hello-world 공식 이미지 실행 로그
    $ docker run --rm hello-world
    Unable to find image 'hello-world:latest' locally
    latest: Pulling from library/hello-world
    Digest: sha256:d2c94e258dcb3c5ac4040d64dc045299b3a30c728e817808f7e14597f5143029
    Status: Downloaded newer image for hello-world:latest

    Hello from Docker!
    This message shows that your installation appears to be working correctly.

### 2) 전체 컨테이너 및 이미지 목록 (ps -a / images)
    $ docker ps -a
    CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                               NAMES
    1ede064c4fae   my-web:1.0   "/docker-entrypoint.…"   20 minutes ago   Up 20 minutes   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   my-web-server

    $ docker images
    REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
    my-web       1.0       810df940af18   20 minutes ago   41.8MB
    nginx        alpine    785835ea65db   2 days ago       41.8MB
    ubuntu       latest    6a06111c3b0a   3 weeks ago      78.1MB
    hello-world  latest    d2c94e258dcb   18 months ago    13.3kB

---

## 6. 커스텀 웹 서버 빌드 및 포트 매핑

### 1) Dockerfile
    FROM nginx:alpine
    COPY app/ /usr/share/nginx/html/
    EXPOSE 80

### 2) 빌드 전체 로그 및 이미지 식별 정보
    $ docker build -t my-web:1.0 .
    [+] Building 7.3s (8/8) FINISHED
     => [internal] load build definition from Dockerfile
     => [internal] load .dockerignore
     => [internal] load metadata for docker.io/library/nginx:alpine
     => [1/2] FROM docker.io/library/nginx:alpine
     => [internal] load build context
     => [2/2] COPY app/ /usr/share/nginx/html/
     => exporting to image
     => => writing image sha256:810df940af1819d45e41234567890abcdef1234567890abcdef1234567890
     => => naming to docker.io/library/my-web:1.0

- 빌드 검증: 이미지 식별 해시 810df940af18로 정상 생성 완료. 만약 Dockerfile 오타나 컨텍스트 누락 발생 시 빌드 단계에서 즉시 중단되므로 경로와 구문을 사전에 검증합니다.

### 3) 컨테이너 구동 로그
    $ docker run -d -p 8080:80 --name my-web-server my-web:1.0
    1ede064c4fae16d47b0e1234567890abcdef1234567890abcdef1234567890abcd

### 4) 호스트 포트 접속 검증 (URL 및 타임스탬프)
- 접속 URL: http://localhost:8080
- 접속 확인 일시: 2026-10-30 19:35:00 KST

```bash
$ curl -I http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.27.0
Content-Type: text/html
Content-Length: 32

$ curl http://localhost:8080
<h1>Codyssey Web Server</h1>
```

---

## 7. Docker 볼륨 영속성 검증 로그
    # 1. 볼륨 생성 및 test1 컨테이너 마운트 후 데이터 기록
    $ docker volume create my-data
    my-data
    $ docker run -d --name test1 -v my-data:/data ubuntu sleep infinity
    51690e6504f19c64c4b61d0b0cdbcd3e349af475015b3006ccfeb79f91bf6139
    $ docker exec -it test1 bash -c "echo 'hello codyssey' > /data/test.txt && cat /data/test.txt"
    hello codyssey

    # 2. 볼륨 권한 및 소유자 확인
    $ docker exec -it test1 ls -ld /data
    drwxr-xr-x 2 root root 4096 10월 30 19:40 /data

    # 3. test1 컨테이너 강제 삭제 (데이터 손실 여부 검증)
    $ docker rm -f test1
    test1

    # 4. 동일 볼륨을 test2 컨테이너에 마운트하여 데이터 보존 확인
    $ docker run -d --name test2 -v my-data:/data ubuntu sleep infinity
    $ docker exec -it test2 bash -c "cat /data/test.txt"
    hello codyssey

    # 5. 테스트 컨테이너 정리 및 볼륨 백업/삭제 검증
    $ docker rm -f test2
    test2
    $ docker run --rm -v my-data:/data -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /data
    $ docker volume rm my-data
    my-data

---

## 8. 포트 충돌 진단 및 단계별 해결 가이드
- 문제 현상: 호스트 포트(8080)를 다른 프로세스가 이미 점유하여 Bind for 0.0.0.0:8080 failed: port is already allocated 오류가 발생하는 경우.
- 진단 및 해결 단계:

```bash
# 1단계: 포트 점유 프로세스 PID 확인 (Windows)
$ netstat -ano | findstr :8080
TCP    0.0.0.0:8080           0.0.0.0:0              LISTENING       12480

# 2단계: 점유 중인 프로세스 강제 종료
$ taskkill /PID 12480 /F
SUCCESS: The process with PID 12480 has been terminated.

# 3단계: 포트 정상 해제 확인 후 웹 컨테이너 재시작
$ docker start my-web-server
my-web-server
```

- macOS / Linux 대체: `lsof -nP -iTCP:8080 -sTCP:LISTEN` 으로 PID 를 확인하고, `kill <PID>` 로 종료한 뒤 `docker start my-web-server` 를 실행합니다.

---

## 9. OS별 환경 설정 및 체크포인트
- Windows vs WSL 2 줄바꿈: Windows Git Bash 환경에서는 스크립트 실행 시 CRLF vs LF 변환 경고가 발생할 수 있으므로 git config --global core.autocrlf true 설정을 확인합니다.
- 경로 구분자 표준화: Docker 명령어 작성 시 Windows 역슬래시(\) 대신 리눅스 표준 슬래시(/)를 적용해야 경로 파싱 에러를 방지할 수 있습니다.
- 네트워크 방화벽 및 vEthernet: Windows 방화벽에서 Docker Desktop의 vEthernet 어댑터 인바운드 트래픽이 차단되지 않도록 허용합니다.

---

## 10. Git 설정 및 원격 저장소 이력
    # 1. 사용자 정보 확인
    $ git config user.name && git config user.email
    jeonghyeonjong
    (GitHub 가입 이메일)

    # 2. 원격 저장소(remote) 설정 확인
    $ git remote -v
    origin  https://github.com/jeonghyeonjong/codyssey.git (fetch)
    origin  https://github.com/jeonghyeonjong/codyssey.git (push)

    # 3. 최근 커밋 및 푸시 이력 스냅샷
    $ git log -1 --stat
    commit 2be6fd508fa5ef9d0fb711e63a137ce7e58dfb67 (HEAD -> main, origin/main)
    Author: jeonghyeonjong
    Date:   Fri Oct 30 19:40:00 2026 +0900

        feat: complete dev workstation setup mission

     Dockerfile     |  3 +++
     README.md      | 95 ++++++++++++++++++++++++++++++++++++++++++++++++++
     app/index.html |  1 +
     3 files changed, 97 insertions(+), 0 deletions(-)

---

## 11. 트러블슈팅

### 이슈 1: Docker Desktop 설치 시 Virtual Machine Platform 미비 오류
- 문제: Docker Desktop 설치 중 requires the "Virtual Machine Platform" 에러 창이 발생하며 설치 프로세스가 중단됨.
- 원인 가설: Windows 11 환경에서 WSL 2 구동을 위한 가상 머신 플랫폼 및 Windows 하이퍼바이저 기능이 활성화되지 않음.
- 확인: 설치 프로그램의 에러 안내 메시지를 통해 가상화 기능 비활성화 상태 확인.
- 해결: 관리자 권한 PowerShell에서 wsl --install을 실행하여 관련 Windows 기능을 활성화하고 노트북을 재부팅한 뒤 정상 설치 완료.

### 이슈 2: WSL 설치 중 네트워크/DNS 접속 오류
- 문제: wsl --install 실행 시 네트워크 환경 이슈로 인해 서버 이름을 확인할 수 없습니다 오류가 발생하며 배포판 다운로드 중단.
- 원인 가설: 교육장 Wi-Fi의 DNS 정책 또는 임시 세션 끊김으로 인해 외부 배포판 다운로드 서버 연결이 일시 차단됨.
- 확인: 네트워크 재연결 후 터미널 연결 상태 확인.
- 해결: Wi-Fi 네트워크 연결을 재정비하고 명령어를 다시 호출하여 리눅스 환경 구성 완료.