# pfSense + WireGuard + Bastion(192.168.1.50) 설정 README

> 목표: 외부에서 WireGuard로 pfSense에 접속 → 배스천 VM(192.168.1.50) 만 접근(Bastion Only).
> 
> 
> 환경: pfSense **WAN이 사설 IP**(더블 NAT) → **Block private networks… 해제** 필요.
> 

---

## 0) 사전 준비

- (있다면) **앞단 공유기/모뎀**에 포트포워딩: **UDP 51820 → pfSense의 WAN IP**
- 배스천 VM: `192.168.1.50/24`, GW/DNS = `192.168.1.1(pfSense)`

---

## 1) WAN 인터페이스 설정

**Interfaces → WAN → Reserved Networks**

- ✅ **Block bogon networks**: **체크 유지**
- ❌ **Block private networks and loopback addresses**: **체크 해제**
    - 이유: pfSense **WAN이 사설 대역(10.x/172.16–31.x/192.168.x)** 이면, 정상 포워딩 트래픽도 사설 SRC로 보이기 때문에 이 옵션이 **정상 트래픽까지 차단**함.
- **Save → Apply**

---

## 2) WireGuard 터널/피어

**VPN → WireGuard → Tunnels → Add**

- **Address/Assignment**: `172.16.20.1/24`
- **Listen Port**: `51820`

**VPN → WireGuard → Peers → Add** (클라이언트마다 1개)

- **Public Key**: (클라이언트 공개키)
- **Allowed IPs (server view)**: 해당 클라 **/32** (예: `172.16.20.2/32`)
    
    *(피어 여러 명일 때 충돌 방지. 단일 환경이면 `172.16.20.0/24`도 동작)*
    
- **Persistent Keepalive**: `25`

---

## 3) WG 인터페이스 활성화

**Interfaces → Assignments**

- `wg0` 추가 → **Enable**(설명: WireGuard) → **Save/Apply**

---

## 4) 방화벽 규칙

### 4-1) WAN 탭

**Firewall → Rules → WAN → Add**

- `Pass | Protocol: UDP | Source: any | Destination: WAN address | Dst Port: 51820 | Log: ✓`
- (**Block bogon** 기본 규칙은 그대로 두기)

### 4-2) WireGuard 탭 (Bastion Only)

**Firewall → Rules → WireGuard**

1. `Pass | Proto: any | Source: WireGuard net | Destination: 192.168.1.50 | Desc: WG→Bastion`
2. *(선택)* `Block | Proto: any | Source: WireGuard net | Destination: LAN net | Desc: WG 나머지 차단`

> LAN 전체 허용이 필요하면 1) 대신
> 
> 
> `Pass | Source: WireGuard net | Destination: LAN net` 한 줄만.
> 

---

## 5) (선택) 인터넷 우회도 필요하면 NAT

**Firewall → NAT → Outbound**: **Hybrid**

**Add**

- **Interface**: WAN
- **Source**: `172.16.20.0/24`
- **Translation / Address**: **WAN address**
    
    **Save/Apply**
    
    *(배스천만 접근이면 생략 가능)*
    

---

## 6) 클라이언트 설정 예시 (Windows/Mac/모바일 WireGuard)

### A) Bastion 전용

```
[Interface]
PrivateKey = <client_private>
Address    = 172.16.20.2/32
DNS        = 192.168.1.1

[Peer]
PublicKey  = <pfSense_public>
Endpoint   = <공인IP또는DDNS>:51820
AllowedIPs = 172.16.20.0/24, 192.168.1.50/32
PersistentKeepalive = 25

```

### B) LAN 전체 접근

```
AllowedIPs = 172.16.20.0/24, 192.168.1.0/24

```

---

## 7) 동작 확인 체크리스트

- **Status → WireGuard**: *Latest Handshake* 가 갱신되는지
- **Status → System Logs → Firewall (Live View)**: `UDP 51820` **Pass** 로그 보이는지
- **Diagnostics → Ping**: *Source = WireGuard 인터페이스* → `192.168.1.50` 응답 OK

---

## 8) 자주 묻는 점

- **“Block private networks…” 왜 해제?**
    
    pfSense **WAN이 사설 IP**일 때 정상 패킷의 출발지가 사설로 보이므로, 해당 옵션이 **정상 트래픽까지 차단**함. 공인 IP를 직접 받는 구조(브리지/패스스루)면 다시 **켜도** 됨.
    
- **Destination “WAN address” vs “This firewall”**
    
    단일 WAN이라면 **WAN address**가 가장 명확. VIP/멀티WAN이면 **This firewall** 사용.
    

---

## 9) 문제 발생 시 빠른 점검

1. 상단 시계 **UTC 표기**면 헷갈릴 수 있음 → *System → General Setup → Time zone* 설정
2. 포트포워딩(앞단 장비) **UDP 51820 → pfSense WAN** 확인
3. WireGuard 피어 **공개키/AllowedIPs** 오탈자 확인
4. 핸드셰이크가 안 되면 **MTU 1420/1380** 시도, 클라 `PersistentKeepalive=25` 설정

---

### 요약

- **WAN이 사설 IP**라면 **Block private networks… 해제**가 핵심.
- **WAN 규칙**: UDP 51820 → **WAN address** 허용.
- **WG 탭**: **192.168.1.50만 허용**(권장) 또는 **LAN net** 전체 허용 중 선택.
- 클라이언트 `AllowedIPs` 로 **Bastion만 / LAN 전체**를 선택적으로 라우팅.
