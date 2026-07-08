# Docker 완전 정복: Chapter 7-5. Docker Networking 🌐

이번 챕터에서는 도커 컨테이너들이 어떻게 서로 통신하고, 외부 세계(인터넷)와 연결되는지 **Docker Networking**의 핵심 아키텍처를 파헤쳐 봅니다.

단순한 명령어 암기를 넘어, **실무 인프라 환경에서 마이크로서비스(MSA)를 설계할 때 네트워크를 어떻게 분리하고 구성하는지** 깊이 있고 전문적인 관점에서 완벽하게 정리해 드리겠습니다.

---

## 🏗️ 1. 도커의 3가지 기본 네트워크 모드

도커를 설치하면 기본적으로 3개의 네트워크가 자동 생성됩니다. (`docker network ls` 로 확인 가능)

### 💡 [Q&A] Bridge 모드와 Host 모드는 정확히 어떻게 다르고 실무에서 어떻게 쓰이나요?
수강생님의 질문처럼 "Bridge 모드면 외부에서 내 컴퓨터로 접근이 가능하다는 말인가요?" 라는 의문이 들 수 있습니다. 정답은 **"기본적으로는 접근 불가능하며, 수동으로 문을 열어줘야(포트 포워딩) 가능합니다"** 입니다. 
두 모드의 차이를 직관적으로 이해하기 위해 **'아파트(Bridge)'와 '단독주택(Host)'**으로 비유해 보겠습니다.

**1) Bridge 모드 (아파트 입주)**
* **원리:** 컨테이너가 호스트 서버(아파트 건물) 안의 101호, 102호처럼 자기만의 독립된 방(격리된 네트워크)을 배정받습니다. 
* **외부 접근:** 외부인이 101호(컨테이너)에 바로 들어갈 수 없습니다. 반드시 아파트 1층 경비실을 거쳐야 합니다. 즉, 외부에서 접속하려면 엔지니어가 `docker run -p 8080:80` (경비실 8080번 문으로 오면 101호 80번 문으로 안내해라)처럼 **포트 포워딩(Port Forwarding)으로 문을 뚫어줘야만 비로소 외부에서 접근 가능**해집니다.
* **실무 활용:** 실무에서 99%의 컨테이너는 이 모드로 띄웁니다. 웹 서버, DB 서버 등 여러 개의 컨테이너를 한 서버에 안전하게 격리해서 띄울 수 있기 때문입니다.

**2) Host 모드 (단독주택 입주)**
* **원리:** 컨테이너가 독립된 방(격리)을 포기하고, 호스트 서버(내 컴퓨터)와 **아예 주소와 현관문을 100% 공유**합니다. 
* **외부 접근:** 아파트 경비실(포트 포워딩)을 거칠 필요가 없습니다. 컨테이너가 "나 5000번 포트로 장사할래!" 하고 켜지는 순간, 호스트 서버의 5000번 포트가 즉시 활성화되어 외부에서 곧바로 다이렉트 접속이 가능해집니다.
* **실무 활용:** 경비실을 거치는 시간조차 아까울 때 사용합니다. 실무에서 초당 수만 건의 트래픽을 처리해야 하는 **초고성능 API 게이트웨이(예: Nginx, HAProxy)**를 운영하거나, 호스트 서버 자체의 네트워크 패킷을 감시하는 **모니터링 에이전트(예: Prometheus Node Exporter)**를 띄울 때 사용합니다. 단, 호스트의 5000번 포트는 1개뿐이므로 똑같은 컨테이너를 2개 띄우면 포트 충돌이 나는 치명적인 단점이 있습니다.

**[Bridge vs Host 모드 직관적 비교 시각화]**
```mermaid
graph TD
    subgraph "Bridge 모드 (아파트)"
        HostB["Docker Host (IP: 192.168.1.10)"]
        PortFW["포트 포워딩<br>-p 8080:80"]
        
        subgraph "격리된 내부 네트워크 (172.17.x.x)"
            ConB["Web Container<br>내부 포트 80"]
        end
        
        HostB --> PortFW
        PortFW -. "변환" .-> ConB
    end
    
    subgraph "Host 모드 (단독주택)"
        HostH["Docker Host (IP: 192.168.1.10)"]
        ConH["Web Container<br>포트 5000 다이렉트 개방"]
        
        HostH === ConH
        note["격리 벽이 없고 네트워크 100% 공유"] -.-> ConH
    end
    
    User1(("외부 사용자")) -- "접속: 192.168.1.10:8080" --> HostB
    User2(("외부 사용자")) -- "접속: 192.168.1.10:5000" --> HostH
    
    style PortFW fill:#ffe0b2,stroke:#ef6c00
    style ConB fill:#bbdefb,stroke:#1565c0
    style ConH fill:#ffccbc,stroke:#d84315
```

### 3) None 모드 (`--network none`)
랜선이 아예 뽑혀 있는 완벽한 **네트워크 단절 상태**입니다. 외부와 통신할 수도 없고, 다른 컨테이너와 통신할 수도 없습니다.
* **실무 활용도:** 외부 해킹의 위험을 100% 차단해야 하는 **초고도 보안 환경(예: 암호화폐 지갑 키 생성기, 폐쇄망 백업 스크립트)**에서 실행 후 즉시 종료되는 배치(Batch) 작업을 돌릴 때 사용합니다.

---

## 🚀 2. [실무 딥 다이브] Default Bridge의 한계와 User-Defined Bridge

강의에서는 컨테이너끼리 통신할 때 내부 IP(`172.17.0.3`)를 쓸 수 있다고 하지만, **실무에서는 내부 IP를 하드코딩해서 통신하는 짓은 절대 금기사항**입니다. 컨테이너가 재시작될 때마다 IP가 랜덤하게 바뀌기 때문입니다.

그럼 어떻게 해야 할까요? IP 대신 **'컨테이너 이름(Name)'**으로 통신해야 합니다. (예: `ping mysql`)
하지만 도커 설치 시 기본 제공되는 기본 `bridge` 네트워크는 치명적인 단점이 있습니다. **컨테이너 이름으로 IP를 찾아주는 DNS 기능이 기본적으로 꺼져 있습니다.**

따라서 실무에서는 무조건 개발자가 직접 **사용자 정의 브릿지(User-Defined Bridge)** 네트워크를 만들어서 사용합니다. 사용자 정의 네트워크 안에서는 도커의 내장 DNS 서버(`127.0.0.11`)가 작동하여, 컨테이너 이름만으로 통신이 가능해집니다.

```bash
# 1. 사용자 정의 네트워크 생성 (실무 표준)
docker network create my-custom-net

# 2. 이 네트워크에 DB와 Web을 소속시킴
docker run -d --name mysql-db --network my-custom-net mysql
docker run -d --name web-server --network my-custom-net nginx
```
이제 `web-server` 안에서 DB에 접속할 때 IP를 몰라도 `mysql-db:3306` 이라는 이름만 적어주면 도커가 알아서 IP를 찾아 연결해 줍니다! (이 원리가 바로 다음 장에서 배울 `docker-compose`의 핵심입니다.)

---

## 🏛️ 3. [실무 설계] 마이크로서비스(MSA) 네트워크 격리 아키텍처

실무에서 대규모 서비스를 구축할 때는 보안을 위해 네트워크를 용도별로 철저히 쪼갭니다(Isolation).
예를 들어 웹 서버(Frontend), 비즈니스 로직(Backend), 데이터베이스(DB)가 있을 때 이를 하나의 네트워크에 묶지 않습니다.

**[실무 MSA 네트워크 격리 시각화]**
```mermaid
graph TD
    subgraph "Docker Host (운영 서버)"
        
        subgraph "Frontend Network (외부 공개용)"
            React["React Web UI"]
        end
        
        subgraph "Backend Network (내부 통신용)"
            Spring["Spring Boot API"]
        end
        
        subgraph "Database Network (철통 보안)"
            MySQL[("MySQL DB")]
            Redis[("Redis Cache")]
        end
        
        React -- "API 호출" --> Spring
        Spring -- "데이터 조회" --> MySQL
        Spring -- "캐시 조회" --> Redis
        
        %% 네트워크 분리를 통한 보안 벽
        React -. "접근 불가 (네트워크 단절)" .-x MySQL
        
    end
    
    User(("일반 사용자")) -- "웹 접속 (Port 80)" --> React
    Hacker(("해커")) -. "DB 직접 해킹 시도" .-x MySQL
    
    style React fill:#bbdefb,stroke:#1565c0
    style Spring fill:#c8e6c9,stroke:#388e3c
    style MySQL fill:#ffccbc,stroke:#d84315
    style Redis fill:#ffccbc,stroke:#d84315
```

**네트워크 설계의 핵심 (Zero Trust):**
1. 사용자는 오직 `Frontend Network`의 React 서버에만 접속할 수 있습니다.
2. React 서버는 `Backend Network`에 속한 Spring 서버와만 통신할 수 있습니다.
3. 오직 Spring 서버만이 `Database Network`에 접근해 DB를 조회할 수 있습니다.
4. **만약 해커가 React 서버를 해킹해서 탈취하더라도, React 컨테이너는 Database 네트워크와 랜선 자체가 단절되어 있으므로 절대로 DB를 해킹할 수 없습니다.**

실무에서는 이처럼 `docker network create frontend-net`, `docker network create backend-net` 등으로 네트워크를 세밀하게 분리하여 아키텍처의 보안과 안정성을 극대화합니다.

---

## 🛠️ 4. 도커 네트워킹의 숨겨진 기술 (Under the Hood)

도커는 도대체 리눅스 안에서 어떻게 가상의 네트워크를 만들어 내는 걸까요? 이 마법은 리눅스 커널의 두 가지 핵심 기술로 구현됩니다.

### 1) Network Namespace (네트워크 격리 방)
이전 7-2 챕터에서 배운 PID Namespace(프로세스 격리)와 유사합니다. 리눅스 커널은 컨테이너가 뜰 때마다 **완전히 독립된 '네트워크 방(Network Namespace)'**을 만들어 줍니다. 이 방 안에는 자신만의 고유한 IP 주소, 라우팅 테이블, 방화벽(iptables) 규칙이 존재합니다.

### 2) Veth Pairs (가상 이더넷 랜선)
각각의 방(컨테이너)이 격리되어 있다면 서로 통신은 어떻게 할까요? 리눅스는 **Virtual Ethernet (veth) Pair**라는 가상의 랜선 1쌍을 제공합니다.
* 랜선의 한쪽 끝(`eth0`)은 컨테이너 방 안쪽 벽에 꽂습니다.
* 랜선의 반대쪽 끝(`veth-xxx`)은 도커 호스트(서버)에 있는 가상의 공유기(`docker0` 브릿지)에 꽂습니다.

**[Veth Pair 작동 원리 시각화]**
```mermaid
graph LR
    subgraph "Docker Host (리눅스 서버)"
        Bridge["docker0 (가상 공유기)<br>IP: 172.17.0.1"]
        
        subgraph "Container 1 (격리된 방)"
            Eth1["eth0<br>IP: 172.17.0.2"]
        end
        
        subgraph "Container 2 (격리된 방)"
            Eth2["eth0<br>IP: 172.17.0.3"]
        end
        
        Bridge == "Veth Pair (가상 랜선)" === Eth1
        Bridge == "Veth Pair (가상 랜선)" === Eth2
    end
    
    style Bridge fill:#e1bee7,stroke:#6a1b9a
    style Eth1 fill:#c8e6c9,stroke:#388e3c
    style Eth2 fill:#c8e6c9,stroke:#388e3c
```
이 가상의 랜선(Veth Pair)을 통해 컨테이너 안에서 보낸 데이터 패킷이 공유기(`docker0`)를 타고 다른 컨테이너로 가거나, 공유기를 거쳐 인터넷 밖으로 나가게 되는 완벽한 물리적 네트워크 모사가 이루어집니다.

---

## 💡 요약
* **Bridge:** 기본 랜선 연결 모드. 실무에서는 기본 브릿지 대신 반드시 **사용자 정의 브릿지(User-Defined Bridge)**를 만들어 컨테이너 이름(DNS)으로 통신한다.
* **Host:** 네트워크 격리를 부수고 서버의 포트를 직결하는 초고성능 모드.
* **None:** 해킹을 원천 차단하는 완전 격리 폐쇄망 모드.
* 실무 아키텍처는 네트워크를 논리적으로 쪼개어 컨테이너 간의 불필요한 접근을 막는 **네트워크 격리(Isolation)**를 통해 보안을 달성한다.
