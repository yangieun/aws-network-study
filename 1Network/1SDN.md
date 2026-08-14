# SDN(Software-Defined Networking) 개념 정리

> **요약**
> SDN은 **Control Plane(제어부)과 Data Plane(전송부)을 논리적으로 분리**하고, 네트워크의 제어 기능을 **논리적으로 중앙화된 Controller에서 소프트웨어로 제어·프로그래밍할 수 있게 하는 네트워크 구조**이다.
> 기존 네트워크가 각 장비의 Control Plane에서 분산적으로 네트워크를 제어했다면, SDN은 **네트워크 전체 관점에서 정책을 정의하고 각 Data Plane이 실행할 동작으로 변환**할 수 있다는 것이 핵심이다.

---

## 1. 먼저 알아야 할 Control Plane과 Data Plane

라우터가 하는 일을 크게 두 가지로 나눌 수 있다.

### Control Plane — "어디로 보낼지 결정"

Control Plane은 **패킷을 어떻게 전달할 것인지 결정하기 위한 정보를 학습·계산하고, Data Plane이 사용할 forwarding 정보를 준비**한다.

예를 들어 OSPF를 사용하는 라우터라면:

```text
OSPF Neighbor 형성
        ↓
네트워크 정보 교환
        ↓
최적 경로 계산
        ↓
Data Plane이 사용할
Forwarding 정보 준비
```

즉, **경로를 결정하는 영역**이다.

### Data Plane — "실제로 패킷을 전달"

Data Plane은 Control Plane이 준비한 정보를 이용하여 **실제로 들어오는 패킷을 처리하고 전달**한다.

```text
패킷 수신
   ↓
이미 준비된 Forwarding 정보 확인
   ↓
출력 Interface 결정
   ↓
실제 패킷 전송
```

따라서 패킷 하나가 들어올 때마다 OSPF가 다시 최적 경로를 계산하는 것이 아니다.

**Control Plane이 미리 결정 → Data Plane이 반복 실행**하는 구조다.

---

# 2. 우리가 헷갈렸던 Routing / Next-hop / Interface

이 부분을 꽤 오래 이야기했기 때문에 예제로 정확하게 정리해두자.

다음과 같은 네트워크가 있다고 하자.

```text
PC-A                    Router A                    Router B
10.10.10.1        G0/1 10.10.20.254 ───── 10.10.20.253
                                                    │
                                                    │
                                               10.10.30.0/24
                                                    │
                                              PC-B 10.10.30.1
```

Router A의 라우팅 테이블에 다음 정보가 있다고 하자.

```text
10.10.10.0/24 → Connected → G0/0
10.10.20.0/24 → Connected → G0/1
10.10.30.0/24 → Next-hop 10.10.20.253 → G0/1
```

PC-A가 `10.10.30.1`로 패킷을 보내면 Router A는 다음과 같이 처리한다.

### ① 목적지 네트워크 확인

```text
DIP = 10.10.30.1

↓ Routing Table 조회

10.10.30.0/24 경로 존재
```

### ② Next-hop 확인

```text
10.10.30.0/24
      ↓
Next-hop = 10.10.20.253
```

여기서 **Next-hop은 최종 목적지가 아니다.**

`10.10.20.253`은 Router B의 인터페이스 IP이다.

### ③ 어떤 Interface로 나갈지 결정

Router A의 G0/1에는:

```text
G0/1 = 10.10.20.254/24
```

가 설정되어 있다.

따라서 Router A는 자신의 인터페이스 정보를 통해:

```text
G0/1 = 10.10.20.254/24

→ 직접 연결 네트워크
→ 10.10.20.0/24
```

임을 알고 있다.

Next-hop인 `10.10.20.253`이 `10.10.20.0/24`에 포함되므로:

```text
Next-hop 10.10.20.253
        ↓
10.10.20.0/24
        ↓
G0/1
```

로 나가게 된다.

### ⭐ 여기서 헷갈렸던 부분

> **Next-hop `10.10.20.253` 자체에 `/24`가 붙어 있어야 하는 것은 아니다.**

Router A가 **자신의 인터페이스 IP + Subnet Mask**를 통해 직접 연결된 네트워크를 이미 알고 있기 때문이다.

그리고:

> `G0/1 = 10.10.20.254/24`

라는 정보 자체는 **인터페이스가 가지고 있는 설정 정보**이다.

이 정보로부터:

```text
10.10.20.0/24 → Connected → G0/1
```

이라는 Connected Route가 만들어진다.

---

# 3. Routing과 ARP의 역할

이 부분도 처음에 혼동했다.

### Routing Table

**어느 방향으로 보낼지를 결정한다.**

```text
DIP
 ↓
목적지 Network
 ↓
Next-hop
 ↓
Output Interface
```

### ARP Table

**같은 L2 구간에서 Next-hop IP에 해당하는 MAC 주소를 알아내는 데 사용한다.**

예를 들어:

```text
Next-hop = 10.10.20.253

↓ ARP

10.10.20.253 = Router B G0/0 MAC
```

ARP 캐시에 정보가 없다면 ARP Request를 통해 알아낸다.

> **ARP Table에 같은 서브넷의 모든 IP/MAC 정보가 미리 저장되어 있는 것은 아니다.**

통신 과정에서 알아낸 IP ↔ MAC 매핑이 캐시된다.

---

# 4. 라우터를 통과할 때 IP와 MAC은 어떻게 되는가?

우리가 가장 많이 헷갈렸던 부분 중 하나다.

PC-A:

```text
IP = 10.10.10.1
```

PC-B:

```text
IP = 10.10.30.1
```

이라고 하자.

### PC-A → Router A

```text
SMAC = PC-A MAC
DMAC = Router A G0/0 MAC

SIP = 10.10.10.1
DIP = 10.10.30.1
```

Router A가 패킷을 받고 Router B로 전달하면:

### Router A → Router B

```text
SMAC = Router A G0/1 MAC
DMAC = Router B G0/0 MAC

SIP = 10.10.10.1
DIP = 10.10.30.1
```

즉,

```text
MAC 주소 → 홉마다 변경
IP 주소  → 기본적으로 유지
```

된다. NAT 같은 별도 기능은 제외한 이야기다.

### ⭐ 우리가 헷갈렸던 부분: "DIP가 내 IP가 아닌데 왜 Ethernet Header를 벗기지?"

처음에는:

> "DIP가 Router A의 IP가 아니니까 디캡슐레이션하지 않는 것 아닌가?"

라고 생각했다.

하지만 라우터는 **수신한 Ethernet Header를 처리한 뒤 내부의 IP Header를 확인해야 한다.**

```text
Ethernet Frame 수신
       ↓
Ethernet Header 처리
       ↓
IP Header 확인
       ↓
DIP = 10.10.30.1
       ↓
내 IP가 아님
       ↓
Routing 수행
       ↓
다음 홉에 맞는 새로운 Ethernet Header 생성
```

그래서 **L2 Ethernet Header는 홉마다 새로 만들어진다.**

그리고 IP Header에서도 완전히 아무것도 안 바뀌는 것은 아니다.

대표적으로:

```text
TTL 64
 ↓ Router
TTL 63
 ↓ Router
TTL 62
```

처럼 **TTL이 라우터를 통과할 때 감소**한다.

Metric과 혼동하면 안 된다.

* **TTL** → 개별 IP 패킷에 존재하며 홉마다 감소
* **Metric** → 라우팅 프로토콜이 경로를 선택할 때 사용하는 값

---

# 5. 전통적인 네트워크 구조

전통적인 네트워크에서는 각 장비가 자신의 Control Plane과 Data Plane을 가지고 있다.

```text
R1              R2              R3
┌───────┐      ┌───────┐      ┌───────┐
│  CP   │      │  CP   │      │  CP   │
│   ↓   │      │   ↓   │      │   ↓   │
│  DP   │      │  DP   │      │  DP   │
└───────┘      └───────┘      └───────┘
```

라우터가 100대라면 **각 라우터에 Control Plane이 존재**한다고 볼 수 있다.

예를 들어 OSPF를 사용한다면 각 라우터가 네트워크 정보를 교환하고 자신의 경로를 계산한다.

---

# 6. ⭐ "SDN 이전에는 모든 경로를 수동 설정했다"는 것은 틀리다

이 부분은 공부하면서 우리가 직접 반례를 발견했다.

SDN 이전에도 당연히:

* OSPF
* IS-IS
* BGP

같은 동적 라우팅 프로토콜이 존재했다.

예를 들어 링크가 장애 나면:

```text
R1 ─ R2 ─X─ R3
 \          /
  ─── R4 ──
```

관리자가 모든 라우터에 접속해서 경로를 다시 입력하는 것이 아니다.

**OSPF가 토폴로지 변화를 감지하고 각 Control Plane에서 경로를 다시 계산할 수 있다.**

따라서:

> ❌ 기존 네트워크는 모든 경로를 사람이 수동으로 설정했다.

가 아니다.

---

# 7. 그렇다면 기존 네트워크의 문제는 무엇이었나?

기존 라우팅 프로토콜은 **네트워크 상태에 따라 경로를 계산하는 것**에는 강하다.

하지만 운영자의 **정책(Policy)**을 네트워크 전체에 적용하는 것은 다른 문제다.

예를 들어:

> "개발팀의 HTTPS 트래픽은 반드시 Firewall을 거쳐 서버망으로 보내라."

이것은 단순히:

> "서버망까지 최적 경로를 계산해."

라는 문제가 아니다.

기존 네트워크에서도 ACL, PBR, QoS 등의 여러 기능으로 정책을 구현할 수 있다.

문제는 Control Plane이 각 장비에 분산되어 있기 때문에 **네트워크 전체에 일관된 정책을 적용하려면 관련 장비들의 설정과 동작을 개별적으로 고려해야 하는 경우가 많다는 것**이다.

우리가 만든 문장으로 표현하면:

> **OSPF 같은 기존 동적 라우팅 프로토콜은 네트워크 상태에 따른 경로 계산을 자동화할 수 있었지만, Control Plane이 각 장비에 분산되어 있었기 때문에 네트워크 전체에 특정 정책을 일관되게 적용하려면 관련 장비들을 개별적으로 구성·관리해야 하는 어려움이 있었다.**

---

# 8. SDN의 핵심 아이디어

SDN에서는 **Control Plane과 Data Plane을 논리적으로 분리**한다.

단순화하면:

```text
             Controller
           Control Plane
                │
          제어 정보/정책
        ┌───────┼───────┐
        ↓       ↓       ↓
       R1      R2      R3
       DP      DP      DP

       실제 사용자 트래픽
       ─────────────────→
```

여기서 매우 중요하다.

### Controller는 중앙에 있는 "라우터"가 아니다

Controller는 **네트워크 제어 기능을 수행하는 소프트웨어**이다.

그리고 당연히 소프트웨어이므로 실제 하드웨어 위에서 실행된다.

예를 들어:

```text
┌──────── Server C ────────┐
│ CPU / RAM                │
│ Linux                    │
│                          │
│ SDN Controller Software  │
└──────────────────────────┘
```

처럼 별도의 서버에서 실행될 수 있다.

### ⭐ 실제 사용자 패킷이 Controller를 거치는 것은 아니다

우리가 처음에 Hub-Spoke처럼 생각했던 부분이다.

```text
❌ 잘못된 이해

R1 → Controller → R2 → Controller → R3
```

가 아니다.

개념적으로:

```text
제어 정보

Controller
 ↓     ↓     ↓
R1    R2    R3


실제 사용자 패킷

R1 → R2 → R3
```

이다.

---

# 9. "논리적 중앙화"란?

SDN에서는 Control Plane을 **논리적으로 중앙화**한다.

전통적인 네트워크:

```text
R1 → 자기 Control Plane
R2 → 자기 Control Plane
R3 → 자기 Control Plane
```

SDN:

```text
         Controller
       네트워크 제어
       ↓     ↓     ↓
      R1    R2    R3
      DP    DP    DP
```

여기서 **논리적 중앙화 = 물리 서버 한 대**라는 뜻은 아니다.

여러 Controller 인스턴스가:

```text
Server 1
Server 2  → 하나의 논리적인 Controller 시스템
Server 3
```

처럼 동작할 수도 있다.

즉 핵심은 **물리적으로 어디에 있느냐가 아니라 네트워크의 제어 기능이 논리적으로 중앙화되어 있다는 것**이다.

---

# 10. Policy(What)와 Forwarding Rule(How)

이 구분도 중요했다.

관리자가 원하는 정책:

> **"개발팀의 HTTPS 트래픽은 Firewall을 거쳐 서버망으로 보내라."**

이것은 네트워크 전체 관점의 **What**이다.

Controller가 토폴로지를 보고:

```text
개발팀
  ↓
 R1
  ↓
 R4
  ↓
Firewall
  ↓
 R7
  ↓
서버망
```

이라는 경로를 결정했다고 하자.

그러면 각 Data Plane에는 자기 역할에 맞는 구체적인 forwarding 동작이 필요하다.

```text
R1
IF 개발팀 HTTPS 트래픽
THEN R4 방향 Port로 전달

R4
IF 개발팀 HTTPS 트래픽
THEN Firewall 방향 Port로 전달

R7
IF 해당 트래픽
THEN 서버망 방향 Port로 전달
```

따라서 우리가 정리한 표현은:

> **Policy(What)**
> 운영자가 네트워크 전체 관점에서 "어떤 트래픽을 어떻게 처리하고 싶은가"를 정의한다.
>
> **Forwarding Rule(How)**
> 그 정책을 실현하기 위해 각각의 Data Plane이 자기 위치에서 수행해야 할 구체적인 동작이다.

Forwarding Rule은 **Routing Protocol이 아니다.**

OSPF 같은 Routing Protocol은 Control Plane에서 경로 정보를 학습하고 계산하는 역할이다.

---

# 11. SDN에서는 더 세밀한 트래픽 제어가 가능하다

전통적인 IP 라우팅을 단순화하면 주로:

```text
Destination IP
      ↓
어디로 Forwarding?
```

을 중심으로 동작한다.

SDN의 대표적인 forwarding 모델에서는 구현에 따라 여러 필드를 기준으로 트래픽을 구분할 수 있다.

예:

```text
IF
SIP = 개발팀 대역
DIP = 서버망 대역
TCP Dst Port = 443

THEN
특정 Port로 Forwarding
```

즉 단순히 목적지뿐만 아니라:

```text
Source IP
Destination IP
Protocol
TCP/UDP Port
...
```

등을 조건으로 활용하여 더 세밀하게 트래픽을 제어할 수 있다.

이를 흔히 **Match → Action** 방식으로 이해할 수 있다.

단, 모든 SDN이 반드시 이 방식으로만 동작한다고 정의하면 안 된다.

---

# 12. Northbound / Southbound Interface

Controller를 중심으로 위쪽과 아래쪽 인터페이스를 구분한다.

```text
Application / 관리자
        │
        │ Northbound Interface
        ↓
┌───────────────────┐
│  SDN Controller   │
└───────────────────┘
        │
        │ Southbound Interface
        ↓
   Data Plane 장비
```

### Northbound Interface

Controller와 **상위 Application/관리 영역** 사이의 인터페이스.

운영자가 원하는 정책이나 애플리케이션 요구사항을 Controller에 전달하는 데 활용된다.

### Southbound Interface

Controller와 **Data Plane 장비** 사이의 인터페이스.

Controller가 forwarding 동작을 전달하거나 장비의 상태 정보를 수집하는 데 사용된다.

**OpenFlow**는 대표적인 Southbound 프로토콜 중 하나다.

```text
OpenFlow = Southbound의 한 가지 방식

Southbound = OpenFlow
```

라고 동일시하면 안 된다.

---

# 13. OpenFlow와 VXLAN은 다르다

우리가 중간에 혼동했던 부분이다.

### OpenFlow

Controller ↔ Data Plane 사이에서 **제어 정보를 주고받기 위해 사용될 수 있는 대표적인 SDN 프로토콜**이다.

### VXLAN

OpenFlow와 목적이 다르다.

VXLAN은 실제 사용자 트래픽을 **캡슐화하여 Overlay Network를 구성하는 기술**이다.

따라서 지금 단계에서는:

> **OpenFlow → 제어 영역**
>
> **VXLAN → 실제 트래픽을 운반하는 Overlay 영역**

정도로 구분해두면 충분하다.

---

# 14. SDN ≠ 단순 네트워크 자동화

이것도 중요한 차이다.

자동화 Script를 만들어:

```text
       Script
     ↓   ↓   ↓
    R1  R2  R3
    CP  CP  CP
    ↓   ↓   ↓
    DP  DP  DP
```

100대의 라우터 설정을 자동으로 변경한다고 하자.

관리자가 CLI로 100번 입력하는 대신 Script가 대신했을 뿐, **각 라우터의 Control Plane이 독립적으로 존재하는 구조 자체는 그대로**다.

반면 SDN의 핵심은 단순히 설정을 빠르게 배포하는 것이 아니라:

> **Control Plane과 Data Plane을 논리적으로 분리하고 네트워크 제어를 소프트웨어를 통해 논리적으로 중앙화·프로그래밍할 수 있도록 하는 것**

이다.

---

# 15. SDN의 `Software-Defined`란?

전통적인 네트워크에도 당연히 소프트웨어는 있었다.

따라서:

> ❌ 소프트웨어를 사용해서 Software-Defined

가 아니다.

핵심은:

> **네트워크의 동작을 개별 하드웨어 장비의 설정에 강하게 묶어두지 않고, 소프트웨어를 통해 네트워크 전체 관점에서 프로그래밍하고 제어할 수 있도록 한다.**

는 것이다.

즉 관리자의 관점이:

```text
"R1에 이 명령어"
"R4에 이 명령어"
"R7에 이 명령어"
```

에서 점차:

```text
"나는 네트워크가
이 정책대로 동작하기를 원한다."
```

라는 방향으로 올라가는 것이 중요한 변화다.

---

# 최종 정리 — SDN의 정의

**SDN(Software-Defined Networking)**은 **Control Plane과 Data Plane을 논리적으로 분리하고, 네트워크의 제어 기능을 논리적으로 중앙화된 Controller에서 소프트웨어로 프로그래밍할 수 있도록 하는 네트워크 구조/접근 방식**이다.

Controller는 네트워크 전체의 상태와 정책을 바탕으로 네트워크 동작을 결정하고, 이를 각 Data Plane이 실행할 수 있는 forwarding 동작으로 반영한다.

# SDN의 주요 특징

1. **Control Plane과 Data Plane의 논리적 분리**
2. **네트워크 제어의 논리적 중앙화**
3. **Controller를 통한 네트워크 전체 관점의 제어**
4. **소프트웨어를 통한 네트워크 프로그래밍 가능성(Programmability)**
5. **정책(What)을 장비별 동작(How)으로 변환**
6. **Northbound / Southbound Interface를 통한 계층 간 통신**
7. **기존 목적지 기반 라우팅보다 세밀한 트래픽 제어가 가능한 구조**
8. **운영자가 개별 장비 설정보다 네트워크 전체 정책에 집중할 수 있는 구조**
