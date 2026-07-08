# Docker 완전 정복: Chapter 7-4. Demo - Docker Storage 🛠️

이번 데모 실습에서는 이전 챕터(7-3)에서 배운 **Docker Storage 메커니즘이 실제 서버에서 어떻게 동작하는지** 눈으로 확인해 보겠습니다.

> 🚨 **[최신화 가이드]** 강의 영상(2017~2019년 기준)에서는 `aufs` 스토리지를 기준으로 설명하지만, **현재 도커 생태계는 `aufs`를 완전히 퇴출시키고 `overlay2`를 100% 표준으로 사용**합니다. 따라서 본 문서는 철저히 최신 실무 표준인 `overlay2` 기준으로 재작성되었습니다.

---

## 🔍 1. 도커의 저장소 위치 확인 (`/var/lib/docker`)

도커 데몬이 설치된 리눅스 서버에서 도커가 데이터를 저장하는 최상위 디렉토리를 확인해 봅니다.

```bash
# 리눅스 환경의 경우 직접 조회 가능
ls -l /var/lib/docker

# 🚨 Mac 유저의 경우 (Docker Desktop)
# Mac 터미널에서 위 명령어를 치면 "No such file or directory" 에러가 뜹니다.
# 이유는 도커가 Mac 하드디스크가 아닌 '숨겨진 리눅스 가상머신(VM)' 안에 설치되기 때문입니다.
# Mac에서 억지로 이 폴더를 보려면 아래처럼 임시 알파인 리눅스 컨테이너를 띄워서 훔쳐봐야 합니다.
docker run --rm -it -v /var/lib/docker:/var/lib/docker alpine ls -l /var/lib/docker
```
**출력 결과 (예시):**
```text
drwx--x--x  4 root root 4096 containers/
drwx------  3 root root 4096 image/
drwxr-x---  3 root root 4096 network/
drwx--x--- 10 root root 4096 overlay2/   <-- (구 aufs 폴더를 대체하는 최신 표준)
drwx------  4 root root 4096 plugins/
drwx------  2 root root 4096 swarm/
drwx------  2 root root 4096 volumes/
```
* **`containers/`**: 컨테이너의 런타임 정보들이 저장됩니다.
* **`image/`**: 다운받거나 빌드한 이미지들의 메타데이터가 저장됩니다.
* **`volumes/`**: `docker volume create`로 만든 데이터 영구 보존용 물리적 공간이 생기는 곳입니다.
* **`overlay2/`**: 이미지의 실제 레이어 파일(소스코드, OS 등)이 물리적으로 저장되는 핵심 폴더입니다.

---

## ⚙️ 2. 현재 사용 중인 Storage Driver 확인 (`docker info`)

내 서버의 도커가 어떤 스토리지 드라이버를 쓰는지 정확히 확인하려면 `docker info` 명령어를 씁니다.

```bash
docker info | grep -i "Storage Driver"
```
**출력 결과:**
```text
 Storage Driver: overlayfs
```
* **Storage Driver**: 최신 리눅스 커널과 최신 도커 버전에서는 기존의 `overlay2`라는 이름 대신, 그것의 기반이 되는 리눅스 커널 파일 시스템의 공식 명칭인 **`overlayfs`**로 출력되는 경우가 많습니다. (`overlay2`와 `overlayfs`는 사실상 같은 기술을 의미하는 실무 표준입니다.)

---

## 🥞 3. `overlay2` 레이어 구조 파헤치기

강의의 `aufs`는 `diff`, `layers`, `mnt` 폴더를 썼지만, 최신 `overlay2`는 구조가 더 직관적입니다. 
이미지를 하나 다운로드한 뒤 `/var/lib/docker/overlay2` 내부를 살펴보겠습니다.

```bash
# 1. 테스트용 nginx 이미지 다운로드
docker pull nginx

# 2. overlay2 폴더 확인
# (Mac 유저는 아까처럼 임시 컨테이너를 띄워서 확인해야 합니다)
docker run --rm -it -v /var/lib/docker:/var/lib/docker alpine ls -l /var/lib/docker/overlay2
```

### 💡 [Mac 유저 딥 다이브] 에러! `overlay2` 폴더가 없는데요?!
수강생님의 터미널 출력 결과를 보면 `overlay2` 폴더 대신 `buildkit`, `image`, `jfs`, `tmp` 같은 폴더들만 덩그러니 있습니다. 이것은 에러가 아니라 **매우 훌륭한 발견**입니다!

최신 Mac용 Docker Desktop은 과거의 무겁고 투박한 파일 관리 방식을 완전히 버렸습니다. 
* Apple의 최신 가상화 기술인 **VirtioFS (`jfs` 폴더 표기)**를 도입했고,
* 차세대 컨테이너 엔진인 **Containerd Image Store** 기능을 탑재하도록 진화했습니다.

이 때문에 최신 Mac 환경에서는 `overlay2` 폴더를 예전처럼 1차원적으로 노출하지 않고, 내부적으로 `buildkit`이나 `image` 폴더 안쪽(또는 `/var/lib/containerd/`)에 더 꽁꽁 숨겨서 관리합니다. 이렇게 함으로써 **Mac 파일 시스템과의 동기화 속도를 비약적으로 끌어올리고 보안을 극대화**한 것입니다. 
따라서 `overlay2` 폴더가 보이지 않는다면, **"아, 내 맥북의 도커 데스크탑이 최신 차세대 아키텍처(Containerd + VirtioFS)로 훌륭하게 세팅되어 있구나!"** 라고 이해하시면 완벽합니다.

> *(참고로 실무 리눅스 서버(Ubuntu 등)에 직접 도커를 설치하면 여전히 `overlay2` 폴더가 명확하게 보이며, 수많은 해시(Hash)값으로 된 레이어 폴더들이 나옵니다. 그 내부 구조는 아래와 같습니다.)*

특정 폴더 안에 들어가보면 아래와 같은 구조로 되어 있습니다.
* **`diff/` (UpperDir 역할)**: 이 레이어에서 "실제로 추가되거나 변경된 파일들"만 들어있는 폴더입니다.
* **`merged/` (MergedDir 역할)**: (컨테이너가 실행 중일 때만 존재) 밑에 깔린 레이어들과 `diff`가 완벽하게 합쳐져서 컨테이너에게 제공되는 뷰(View)입니다.
* **`l/` (Links)**: 너무 긴 해시 경로를 리눅스 커널이 처리하기 힘들기 때문에 짧은 심볼릭 링크(단축키)를 모아둔 폴더입니다.

---

## ⚡ 4. 이미지 빌드와 캐시(Cache)의 마법

간단한 파이썬 웹 애플리케이션을 빌드해 보겠습니다.

### 💡 [Q&A] `docker build` 명령어 오류 원인 파악하기
수강생님이 `docker build -t simple-webapp .` 명령어를 쳤을 때 난 에러(`failed to read dockerfile`)는 **"현재 터미널이 위치한 폴더(`.`)에 `Dockerfile`이라는 이름의 파일이 없어서 빌드를 시작할 수 없다"**는 뜻입니다.

명령어 맨 끝의 점(`.`)은 "현재 폴더에 있는 Dockerfile을 읽어서 빌드해라"라는 명령어입니다. 따라서 실습을 위해 터미널에서 빈 폴더를 하나 만들고, 그 안에 아주 간단한 Dockerfile을 직접 만들어 주어야 합니다.

**[실습을 위한 준비 세팅]**
```bash
# 1. 실습용 빈 폴더 만들고 들어가기
mkdir my-docker-test && cd my-docker-test

# 2. 간단한 Dockerfile 만들기 (복사해서 터미널에 붙여넣으세요)
cat <<EOF > Dockerfile
FROM ubuntu
RUN apt-get update && apt-get install -y python3
CMD ["echo", "Hello Docker"]
EOF
```

### 첫 번째 빌드 (초기 빌드)
이제 `Dockerfile`이 있는 폴더에서 아래 명령어를 다시 쳐보세요.
```bash
docker build -t simple-webapp .
```
처음 빌드할 때는 베이스 이미지(`ubuntu`)를 다운받고, 파이썬을 설치하느라 시간이 꽤 소요됩니다.

### 두 번째 빌드 (동일한 코드 빌드)
만약 소스코드나 Dockerfile을 단 1글자도 안 고치고 다시 똑같이 빌드를 돌린다면 어떻게 될까요?
```bash
docker build -t simple-webapp-v2 .
```
결과는 **단 1초 만에 완료**됩니다!
빌드 로그를 보면 각 스텝마다 **`CACHED`**라는 단어가 뜹니다. 도커가 "어? 이거 아까 만들었던 레이어랑 완벽히 똑같네? 안 만들어! 기존 거 쓸게!" 하고 **재사용(Cache)**했기 때문입니다.

---

## 💥 5. 캐시 무효화 (Cache Invalidation) 주의점

자, 이번에는 내 애플리케이션의 소스 코드(`app.py`)를 단 한 줄 수정했다고 가정해 봅시다.
그리고 다시 빌드를 돌립니다.

```bash
docker build -t simple-webapp-v3 .
```

도커 파일이 다음과 같다고 가정해 봅시다.
1. `FROM ubuntu` (OS)
2. `RUN apt-get install python3` (설치)
3. `RUN pip install flask` (라이브러리 300MB)
4. **`COPY app.py /app` (소스코드 5KB) 👈 여기서 내용이 변경됨!**
5. `ENTRYPOINT ["python3", "app.py"]`

**[동작 결과]**
* 스텝 1, 2, 3번까지는 이전에 만들어둔 레이어를 **그대로 재사용(Cache)**합니다.
* 하지만 스텝 4번에서 원본 파일이 변경된 것을 도커가 눈치채고, 여기서부터는 캐시를 쓰지 않고 **새로운 레이어를 물리적으로 생성**합니다.
* 🚨 **중요 (실무 포인트):** 4번 레이어가 새로 만들어졌으므로, **그 뒤에 따라오는 5번 레이어도 무조건 새로 만들어집니다.** (5번 코드가 안 바뀌었어도 앞단이 바뀌면 뒷단은 전부 캐시가 깨집니다!)

따라서 실무에서는 Dockerfile을 짤 때, **자주 안 바뀌는 것(OS, 라이브러리 설치)을 위에 적고, 맨날 바뀌는 것(소스코드 COPY)을 맨 밑에 적는 것**이 빌드 속도를 1초 컷으로 유지하는 절대적인 비법입니다.

---

## 📊 6. 이미지 용량의 진실 (`docker system df`)

현재 내 서버에 똑같은 `simple-webapp` 이미지가 v1, v2, v3 세 개나 있습니다.
`docker images`를 쳐봅니다.

```bash
docker images
# simple-webapp-v1 : 450MB
# simple-webapp-v2 : 450MB
# simple-webapp-v3 : 450MB
```
분명 소스코드 5KB만 수정했는데, 이미지가 450MB짜리 3개면 내 하드디스크가 총 **1.35GB**나 잡아먹힌 걸까요?

### 진짜 하드디스크 사용량 확인 (`docker system df`)
도커가 하드디스크를 진짜 얼마나 쓰고 있는지 확인하는 가장 정확한 실무 명령어입니다.

```bash
docker system df -v
```
출력 결과의 **`SHARED SIZE`**와 **`UNIQUE SIZE`** 열을 주목해야 합니다.
* **SHARED SIZE (공유된 크기):** v1, v2, v3가 100% 똑같이 쓰고 있는 OS 및 라이브러리 레이어 공간입니다. (약 449.9MB)
* **UNIQUE SIZE (독자적 크기):** v3에서 수정한 5KB짜리 소스코드 레이어 하나만의 크기입니다.

즉, 내 하드디스크는 1.35GB가 날아간 것이 아니라, **449.9MB + 5KB + 5KB** 정도만 쓰고 있는 것입니다! 이것이 도커 레이어 아키텍처가 보여주는 기적 같은 디스크 절약 기술입니다. 

---
### 🎉 데모 요약
1. 도커의 찐 데이터는 `/var/lib/docker/overlay2`에 저장된다.
2. 2026년 스토리지 드라이버 실무 표준은 무조건 `overlay2`다.
3. 빌드 속도가 빠른 이유는 레이어 캐싱 덕분이며, Dockerfile 최적화가 필수다.
4. `docker images`의 용량 표기에 쫄지 마라. 겹치는 레이어는 하나로 공유(`SHARED SIZE`)되므로 실제 디스크는 안전하다. (`docker system df`로 확인!)
