# 개발 워크스테이션 구축 미션

## 1. 실행 환경
- OS: Windows 11
- Shell: Git Bash
- Container Runtime: Docker Desktop (WSL 2 Backend)
- Version Control: Git for Windows

## 2. 수행 체크리스트
- [x] 터미널 기본 조작 및 권한 변경 실습
- [x] Docker 설치 및 기본 환경 점검
- [x] Dockerfile 기반 커스텀 웹 서버 빌드
- [x] 포트 매핑 접속 검증 (8080:80)
- [x] 볼륨 영속성 검증 (my-data)
- [x] Git 사용자 설정 및 VS Code 연동
- [x] 트러블슈팅 2건 작성

## 3. 터미널 조작 및 권한 실습 로그
### 1) app 디렉토리 권한 설정 (755)
\`\`\`bash
$ chmod 755 app
$ ls -ld app
drwxr-xr-x 1 jeonghyeonjong 197609 0 10월 30 19:27 app/
\`\`\`

### 2) 웹 페이지 내용 작성
\`\`\`bash
$ echo "<h1>Codyssey Web Server</h1>" > app/index.html
\`\`\`

## 4. 커스텀 웹 서버 빌드 및 포트 매핑
### 1) Dockerfile
\`\`\`dockerfile
FROM nginx:alpine
COPY app/ /usr/share/nginx/html/
EXPOSE 80
\`\`\`

### 2) 이미지 빌드 및 컨테이너 실행
\`\`\`bash
$ docker build -t my-web:1.0 .
$ docker run -d -p 8080:80 --name my-web-server my-web:1.0
$ docker ps
CONTAINER ID   IMAGE        COMMAND                  PORTS                               NAMES
1ede064c4fae   my-web:1.0   "/docker-entrypoint.…"   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp   my-web-server
\`\`\`

### 3) 로컬 접속 검증
\`\`\`bash
$ curl http://localhost:8080
<h1>Codyssey Web Server</h1>
\`\`\`

## 5. Docker 볼륨 영속성 검증 로그
\`\`\`bash
# 1. 볼륨 생성 및 test1 컨테이너 마운트 후 데이터 작성
$ docker volume create my-data
$ docker run -d --name test1 -v my-data:/data ubuntu sleep infinity
$ docker exec -it test1 bash -c "echo 'hello codyssey' > /data/test.txt && cat /data/test.txt"
hello codyssey

# 2. test1 컨테이너 강제 삭제
$ docker rm -f test1

# 3. 동일 볼륨을 test2 컨테이너에 마운트하여 데이터 보존 확인
$ docker run -d --name test2 -v my-data:/data ubuntu sleep infinity
$ docker exec -it test2 bash -c "cat /data/test.txt"
hello codyssey

# 4. test2 컨테이너 정리
$ docker rm -f test2
\`\`\`

## 6. Git 사용자 설정
\`\`\`bash
$ git config user.name && git config user.email
(설정된 GitHub 사용자 이름 및 이메일 확인 완료)
\`\`\`

## 7. 트러블슈팅
### 이슈 1: Docker Desktop 설치 시 가상화 플랫폼(Virtual Machine Platform) 미비 오류
- **문제**: Docker Desktop 설치 중 `requires the "Virtual Machine Platform"` 에러가 발생하며 설치가 중단됨.
- **원인 가설**: Windows 11 환경에서 WSL 2 및 가상 머신 플랫폼 기능이 활성화되어 있지 않아 Docker 데몬을 구동하지 못함.
- **확인**: 설치 프로그램 에러 메시지를 통해 가상화 기능 비활성화 상태를 확인.
- **해결**: 관리자 권한 PowerShell에서 `wsl --install`을 실행하여 관련 기능을 활성화하고 재부팅 후 정상 설치 완료.

### 이슈 2: WSL 설치 중 네트워크/DNS 접속 오류
- **문제**: `wsl --install` 실행 시 네트워크 환경 이슈로 인해 배포판 다운로드 서버 주소를 찾지 못하고 중단됨.
- **원인 가설**: 교육장 Wi-Fi의 DNS 또는 보안 정책으로 인해 외부 배포판 다운로드 경로가 일시적으로 차단됨.
- **확인**: 네트워크 재연결 후 터미널 상태 확인.
- **해결**: Wi-Fi 네트워크 연결을 재정비하고 명령어를 다시 호출하여 리눅스 환경 구성 완료.
