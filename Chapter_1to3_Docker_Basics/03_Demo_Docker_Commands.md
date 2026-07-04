# Docker 완전 정복: 3강. 컨테이너 안으로 직접 들어가 보기! 🚪 (Demo - Docker Commands)

지난 2강에서는 컨테이너를 실행(`run`), 확인(`ps`), 중지(`stop`), 삭제(`rm`)하는 기본적인 라이프사이클과 터미널에 뜨는 글자들의 의미를 완벽하게 해독해 보았습니다!

이번 3강(강의의 2.2 파트 데모)에서는 지난번 내용의 복습과 함께, **컨테이너 내부로 직접 로그인해서 들어가는 마법 같은 방법(`-it` 옵션)**과 알아두면 피가 되고 살이 되는 소소한 팁들을 배워보겠습니다.

---

## 1. 이미지의 이름 짓기 규칙: 공식(Official) vs 개인(Custom)

우리가 `docker run ubuntu`나 `docker run centos`처럼 간단한 이름만 적어서 실행할 수 있었던 이유는 무엇일까요?
Docker Hub(이미지들이 모여있는 앱스토어 같은 곳)에는 두 가지 종류의 이미지가 있습니다.

1. **공식 이미지 (Official Images):**
   - Docker 회사에서 공식적으로 검수하고 지원하는 가장 기본적이고 안전한 이미지들입니다.
   - 이름이 아주 깔끔합니다. 예: `ubuntu`, `centos`, `nginx`, `redis`
2. **개인 이미지 (Custom Images):**
   - 저나 여러분 같은 일반 개발자들이 직접 만들어서 올린 이미지들입니다.
   - 반드시 **"만든 사람의 아이디 / 이미지 이름"** 형태로 써야 합니다.
   - 예: `mmumshad/ansible-playable` (강사인 mumshad님이 만든 ansible 이미지)

---

## 2. 드디어 컨테이너 내부로 접속! (`-it` 옵션과 `bash`)

지난 시간에 `ubuntu`나 `centos` 같은 빈 껍데기 운영체제는 켜지자마자 할 일이 없어서 바로 퇴근해버린다(Exited)고 배웠죠?
그렇다면 우리가 **그 빈 방(컨테이너) 안에 직접 들어가서 키보드를 치며 놀고 싶다면** 어떻게 해야 할까요?

이때 등장하는 것이 바로 **대화형 모드(`-it`)** 입니다!

- `-i` (Interactive): 내가 터미널에 치는 입력을 컨테이너로 전달하겠다.
- `-t` (TTY): 컨테이너의 화면을 내 터미널에 보여주겠다.
- **결론:** **`-it`를 쓰면 내 터미널이 컨테이너의 화면으로 완전히 바뀝니다!**

> **항구 비유:** 그동안은 밖에서 컨테이너 박스를 이리저리 옮기기만 했다면, 이제는 **컨테이너 박스의 문을 열고 직접 안으로 걸어 들어가는 것**입니다!

---

## 3. 실습 환경 세팅 및 단계별 실습 가이드

자, 터미널을 열고 이번 데모 영상에서 강사님이 보여준 명령어들을 직접 타이핑하며 체화해 봅시다.

### 💻 실습 1: 컨테이너 내부로 들어가기 (`-it`)

> 🚨 **주의사항 (에러 해결)**
> 만약 `docker run -it centos bash` 를 쳤을 때 `not found` 에러가 발생하셨다면, 최근 Docker Hub 정책 변경으로 인해 기본(`latest`) 버전이 삭제되었기 때문입니다.
> 당황하지 마시고 아래처럼 버전을 명시한 **`centos:7`** 로 실행해 주세요!

```bash
# 1. CentOS 7 버전의 리눅스 운영체제 컨테이너 안으로 접속합니다!
# bash라는 명령어를 추가하여, 컨테이너 안에서 명령어를 칠 수 있는 상태로 만듭니다.
docker run -it centos:7 bash

# --- 실행 결과 화면 (예시) ---
# user@macbook ~ % docker run -it centos:7 bash
# Unable to find image 'centos:7' locally
# 7: Pulling from library/centos
# ... (다운로드 완료) ...
# [root@2d9c8931fd35 /]#
```
*❗ 터미널 앞부분의 글자가 `user@macbook` 에서 갑자기 `[root@2d9c8931fd35 /]#` (임의의 영문숫자 조합) 로 바뀌었나요? 축하합니다! 지금 여러분의 터미널은 내 Mac이 아니라, 격리된 CentOS 컨테이너 내부입니다!*

```bash
# 2. 정말 내가 CentOS 안에 들어왔는지 확인해 봅시다.
# (이 명령어는 리눅스의 버전을 확인하는 명령어입니다.)
cat /etc/*release*

# --- 실행 결과 화면 (예시) ---
# [root@2d9c8931fd35 /]# cat /etc/*release*
# CentOS Linux release 7.9.2009 (AltArch)
# Derived from Red Hat Enterprise Linux 7.8 (Source)
# NAME="CentOS Linux"
# ... (중략) ...
```

```bash
# 3. 방 구경을 다 했으니 밖으로 나갑시다.
exit
```
*(다시 원래의 내 컴퓨터 터미널로 돌아오며, 주인이 나가버린 컨테이너는 할 일이 없어져서 스스로 종료됩니다.)*

---

### 💻 실습 2: 여러 개를 한 번에 삭제 & 강제 종료 코드 (Exit code)
```bash
# 4. 백그라운드(-d)로 2000초 동안 잠만 자는 컨테이너를 실행합니다.
docker run -d centos:7 sleep 2000

# --- 실행 결과 화면 (예시) ---
# user@macbook ~ % docker run -d centos:7 sleep 2000
# 37f98915d4a0e3344ef27f0837a46c93c1a18cfc35aaf5ee3d7c85ac09f15c43
# (아주 긴 고유 ID가 출력됩니다. 이것이 컨테이너의 풀(Full) ID입니다.)
```

```bash
# 5. ps로 돌아가고 있는지 확인합니다.
docker ps

# --- 실행 결과 화면 (예시) ---
# user@macbook ~ % docker ps
# CONTAINER ID   IMAGE      COMMAND        STATUS         NAMES
# 37f98915d4a0   centos:7   "sleep 2000"   Up 5 seconds   confident_dhawan
```

```bash
# 6. 2000초를 기다릴 수 없으니 중간에 강제로 멈춥니다.
# (docker ps에 나온 컨테이너 ID의 앞 3자리 정도만 적어주세요! 예: 37f)
docker stop 컨테이너ID

# --- 실행 결과 화면 (예시) ---
# user@macbook ~ % docker stop 37f
# 37f
```

```bash
# 7. ps -a로 꺼진 기록을 확인해 봅니다.
docker ps -a

# --- 실행 결과 화면 (예시) ---
# user@macbook ~ % docker ps -a
# CONTAINER ID   IMAGE      STATUS                       NAMES
# 37f98915d4a0   centos:7   Exited (137) 7 seconds ago   confident_dhawan
```
*💡 터미널 결과를 보시면 방금 멈춘 컨테이너의 상태가 `Exited (137)`일 것입니다. 정상적으로 일이 다 끝나서 스스로 퇴근한 경우는 `Exited (0)`이지만, 우리가 외부에서 `stop`으로 강제로 목줄을 잡아끈(강제 종료) 경우는 `137`이라는 코드가 남습니다.*

```bash
# 8. 컨테이너 찌꺼기들을 한 번에 청소합니다!
# rm 뒤에 지우고 싶은 컨테이너 ID 앞자리들을 띄어쓰기로 여러 개 적으면 한 번에 지워집니다.
# 예: docker rm a1b 345 e0a
docker rm 컨테이너ID_1 컨테이너ID_2
```

---

### 💻 실습 3: 정말 작고 귀여운 이미지 `busybox` 와 이미지 삭제

Ubuntu나 CentOS는 크기가 좀 큰 편입니다. 아주 잠깐 테스트만 하고 싶을 때 사용하는 초경량 이미지가 있습니다. 바로 `busybox`입니다!

```bash
# 9. busybox 이미지를 다운로드만(pull) 해봅니다.
docker pull busybox

# --- 실행 결과 화면 (예시) ---
# user@macbook ~ % docker pull busybox
# latest: Pulling from library/busybox
# ... (다운로드 완료) ...
# Status: Downloaded newer image for busybox:latest
```

```bash
# 10. 이미지 목록을 확인해서 크기(SIZE)를 비교해 봅니다.
docker images

# --- 실행 결과 화면 (예시) ---
# user@macbook ~ % docker images
# IMAGE              ID             DISK USAGE   CONTENT SIZE   EXTRA
# busybox:latest     fd8d9aa63ba2       6.22MB         1.92MB        
# centos:7           be65f488b776        434MB          108MB    U   
# mysql:8.0          d0304ed9fdb6       1.09GB          245MB    U   
# ... (생략) ...
```
*(CentOS는 100MB가 넘지만, busybox는 겨우 **1.92MB**인 것을 직접 확인하실 수 있습니다!)*

```bash
# 11. 이제 다 쓴 이미지(설계도)를 지워보겠습니다.
docker rmi busybox

# --- 실행 결과 화면 (예시) ---
# user@macbook ~ % docker rmi busybox
# Untagged: busybox:latest
# Deleted: sha256:fd8d9aa63ba2f0982b5304e1ee8d3b90a210bc1ffb5314d980eb6962f1a9715d
# 
# user@macbook ~ % docker images
# IMAGE              ID             DISK USAGE   CONTENT SIZE   EXTRA
# centos:7           be65f488b776        434MB          108MB    U   
# ... (busybox가 목록에서 사라짐)
```
*(단, 2강에서 배웠듯이 어떤 컨테이너가 이 설계도를 바탕으로 만들어져서 남아있다면(꺼져있더라도), `docker rm`으로 컨테이너를 먼저 지워야 `rmi`로 이미지를 지울 수 있습니다!)*

---

## 🎯 학습 점검 및 마무리

오늘은 컨테이너 '밖'에서 조종하는 것을 넘어, **컨테이너 '안'으로 들어가는 법(`-it`)**을 배웠습니다. 

위 실습을 진행해 보시고 다음 사항들을 한 번 점검해 주세요!

* **점검 1:** `docker run -it centos bash`를 쳤을 때 터미널 이름이 `root@...`로 바뀌면서 **컨테이너 내부로 접속되는 것**을 직접 확인하셨나요?
* **점검 2:** `exit`를 쳐서 빠져나온 뒤 `docker ps -a`를 쳤을 때, 해당 컨테이너가 정상적으로 `Exited (0)` 상태가 된 것을 이해하셨나요?

**💡 추가 보충 의견:**
혹시 `-d`(백그라운드에서 실행, 내 터미널은 자유)와 `-it`(포어그라운드에서 실행하고 화면 연결, 내 터미널은 묶임)의 차이가 헷갈리시나요? 둘은 완전히 정반대의 역할을 합니다! 이 부분이 헷갈리신다면 언제든 말씀해 주세요.

내용이 잘 이해되셨고 실습까지 무사히 완료하셨다면, **다음 랩스 파트(2.3 Demo - Docker Labs)**로 넘어가 보겠습니다! 🚀
