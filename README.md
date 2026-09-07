# ABC Tech Network Security Lab

OPNsense 기반 본사-지사 네트워크 보안 구축 및 장애 대응 실습

> VMware 기반 가상 환경에서 진행한 네트워크 보안 실습 프로젝트입니다.  
> WAN은 Bridged 방식으로 외부 네트워크를 구성했으며 실제 운영 장비가 아닌 가상 환경에서 방화벽, NAT, VPN, IPS 기능을 구현하고 검증했습니다.

---

## 1. 프로젝트 개요

기존 Routing/Switching 학습에서 확장하여 OPNsense 방화벽을 이용한 네트워크 보안 환경을 구성했습니다.

본사와 지사 네트워크를 분리하고 Site-to-Site IPsec VPN으로 연결했으며, 본사에서는 Firewall Policy, Source NAT, Suricata IPS를 적용했습니다.

단순히 설정만 완료하는 것이 아니라 실제 트래픽과 로그를 통해 각 기능의 동작 여부를 확인하는 것을 목표로 했습니다.

### 주요 구현 항목

- Firewall Policy
- Source NAT
- Firewall Logging
- Packet Capture
- IKEv2 Site-to-Site IPsec VPN
- Pre-Shared Key Authentication
- Suricata IPS
- ET Open Ruleset
- IPS Alert / Drop
- 장애 원인 확인 및 복구

---

## 2. Network Topology

![Network Topology](01_topology.png)

### IP 구성

| 구분 | HQ | Branch |
|---|---|---|
| WAN | 192.168.219.109 | 192.168.219.110 |
| LAN Gateway | 10.10.10.1/24 | 10.20.10.1/24 |
| LAN Network | 10.10.10.0/24 | 10.20.10.0/24 |
| Test Host | 10.10.10.2/24 | 10.20.10.2/24 |

WAN은 VMware Bridged 방식으로 실제 공유기 네트워크에 연결했습니다.

---

## 3. Firewall Policy

본사 LAN에서 외부 네트워크로 통신할 수 있도록 별도의 Firewall Policy를 구성했습니다.

### Policy

- Interface: LAN
- Direction: In
- IPv4
- Source: HQ LAN
- Destination: Any
- Action: Pass
- Logging: Enabled

정책 이름은 `HQ_LAN_TO_INTERNET`으로 설정했습니다.

![Firewall Policy Log](02_firewall_policy_log.png)

Live View에서 다음 트래픽이 해당 정책에 매칭되는 것을 확인했습니다.

- `10.10.10.2 → 8.8.8.8` ICMP
- `10.10.10.2 → 8.8.8.8:53` DNS

이를 통해 실제 LAN 트래픽이 생성한 Firewall Policy를 통과하는 것을 검증했습니다.

---

## 4. Source NAT

내부 사설 IP를 이용하는 HQ LAN 사용자가 외부 네트워크에 접근할 수 있도록 Source NAT를 적용했습니다.

내부에서 발생한 트래픽:

`10.10.10.2 → 8.8.8.8`

WAN 인터페이스에서 Packet Capture한 결과:

`192.168.219.109 → 8.8.8.8`

로 변환된 것을 확인했습니다.

![Source NAT Packet Capture](03_snat_packet_capture.png)

이를 통해 내부 주소 `10.10.10.2`가 OPNsense WAN 주소 `192.168.219.109`로 변환되어 외부로 전달되는 것을 확인했습니다.

---

## 5. Site-to-Site IPsec VPN

본사와 지사 내부 네트워크를 연결하기 위해 IKEv2 기반 Site-to-Site IPsec VPN을 구성했습니다.

### VPN 구성

| 항목 | 설정 |
|---|---|
| IKE Version | IKEv2 |
| Authentication | Pre-Shared Key |
| Encryption | AES256 |
| Integrity | SHA256 |
| DH Group | 14 |
| HQ Network | 10.10.10.0/24 |
| Branch Network | 10.20.10.0/24 |

HQ와 Branch의 WAN에서는 IKE 및 IPsec 협상을 위해 필요한 트래픽을 허용했습니다.

- UDP 500
- UDP 4500
- ESP

### VPN 상태 확인

![IPsec VPN Status](04_ipsec_vpn_status.png)

Phase 1 연결 상태와 Phase 2의 `INSTALLED` 상태를 확인했으며, 실제 통신 후 Bytes In / Out 값이 증가하는 것도 확인했습니다.

### 터널 통신 검증

HQ의 `10.10.10.1`을 Source로 지정하여 Branch의 `10.20.10.1`까지 Ping 테스트를 진행했습니다.

![IPsec VPN Ping](05_ipsec_vpn_ping.png)

패킷 손실률 `0.0%`를 확인하여 두 내부 네트워크 간 IPsec VPN 통신이 정상적으로 동작하는 것을 검증했습니다.

---

## 6. Suricata IPS

본사 OPNsense에 Suricata 기반 IPS를 구성했습니다.

### IPS 구성

- Capture Mode: Netmap (IPS)
- Interface: LAN
- ET Open Ruleset 사용
- Syslog Alert 활성화
- 테스트 Rule을 이용한 Alert / Drop 검증

Firewall Policy에서 허용된 트래픽이라도 IPS에서 별도로 검사하고 차단할 수 있는지 확인하는 것을 목표로 했습니다.

### IPS 차단 테스트

테스트 규칙을 이용해 다음 DNS 트래픽을 대상으로 검증했습니다.

`10.10.10.2 → 8.8.8.8:53`

처음에는 Alert 방식으로 탐지 여부를 확인한 후 Action을 Drop으로 변경했습니다.

![IPS Block Test](06_ips_block_test.png)

Suricata Alert에서 `blocked` 상태를 확인했으며 Windows에서:

`nslookup google.com 8.8.8.8`

테스트 결과 DNS Request Timeout이 발생하여 실제 패킷이 차단되는 것을 확인했습니다.

---

## 7. Troubleshooting

### 7.1 WAN 연결 및 Default Route 문제

**증상**

OPNsense WAN 인터페이스에 IP가 존재했지만 외부 IP로 통신할 수 없었습니다.

**확인**

- WAN Interface 상태
- VMware Network 구성
- Gateway
- Routing Table

을 순서대로 확인했습니다.

**원인**

초기 VMware NAT 환경과 Gateway 설정이 정상적으로 연결되지 않았으며 Default Route가 정상적으로 설정되지 않은 상태였습니다.

**조치**

WAN 연결을 Bridged 방식으로 변경하고 DHCP Gateway를 정상 Default Gateway로 적용했습니다.

**검증**

OPNsense에서 `8.8.8.8` Ping 결과 Packet Loss `0%`를 확인했습니다.

---

### 7.2 테스트 트래픽이 OPNsense를 우회

**증상**

Windows에서 인터넷 Ping은 성공했지만 OPNsense Firewall/NAT를 실제로 통과한 것인지 확인할 수 없었습니다.

**확인**

`tracert`를 이용해 트래픽 경로를 확인했습니다.

초기에는 실제 PC의 기본 Gateway를 이용해 외부로 직접 통신하고 있었습니다.

**조치**

검증 대상 트래픽이 OPNsense LAN Gateway를 경유하도록 테스트용 Route를 적용했습니다.

**검증**

경로가 다음과 같이 변경되는 것을 확인했습니다.

`10.10.10.1 → 192.168.219.1 → External Network`

이후 Firewall Log와 WAN Packet Capture를 이용해 Policy 및 NAT 적용 여부를 추가로 검증했습니다.

---

### 7.3 IPS Block 로그와 실제 동작 불일치

**증상**

Suricata Alert에서는 `blocked` 상태가 기록됐지만 실제 DNS 통신은 계속 가능했습니다.

**확인**

- Traffic Path
- Suricata Alert
- Netmap IPS Mode
- 실제 DNS 요청 결과

를 비교했습니다.

로그만 보고 차단이 완료됐다고 판단하지 않고 실제 서비스 통신 여부를 다시 확인했습니다.

**원인**

VMware 가상 NIC 환경에서 Netmap native 동작이 정상적으로 패킷을 Drop하지 못하는 문제가 있었습니다.

**조치**

Netmap Emulated Mode를 사용하도록 다음 Tunable을 적용했습니다.

`dev.netmap.admode = 2`

**재검증**

동일한 DNS 요청을 다시 수행한 결과:

`DNS request timed out`

이 발생했고 Suricata Alert에서도 `blocked` 상태를 확인했습니다.

---

## 8. 검증 결과

| 항목 | 결과 |
|---|---|
| HQ LAN → Internet Firewall Policy | 성공 |
| Firewall Policy Log | 성공 |
| Source NAT | 성공 |
| WAN Packet Capture NAT 확인 | 성공 |
| IKEv2 Phase 1 | 성공 |
| IPsec Phase 2 | INSTALLED |
| HQ → Branch VPN 통신 | 성공 |
| Suricata Alert | 성공 |
| Suricata Drop | 성공 |
| 실제 DNS 차단 | 성공 |

---

## 9. 프로젝트를 통해 확인한 점

이번 실습을 통해 Firewall에서 트래픽을 허용하는 것과 IPS가 해당 트래픽을 검사하고 차단하는 것은 서로 다른 단계라는 점을 직접 확인했습니다.

또한 설정 화면이나 로그만 보고 정상 여부를 판단하지 않고, Firewall Log, Packet Capture, VPN 상태, Ping, DNS 요청 등 실제 트래픽을 함께 확인해야 정확하게 장애 원인을 판단할 수 있다는 점을 경험했습니다.

특히 IPS 실습에서는 `blocked` 로그만 확인했을 때는 정상적으로 차단됐다고 생각할 수 있었지만 실제 통신 테스트에서는 트래픽이 계속 전달되고 있었습니다. 이후 가상 NIC와 Netmap 동작을 확인하고 설정을 수정하여 실제 차단까지 재검증했습니다.

---

## 10. Environment

- OPNsense 26.7
- VMware
- Windows Test Host
- Suricata IPS
- ET Open Ruleset

> 본 프로젝트는 네트워크/보안 기술 학습을 위해 VMware 기반 가상 환경에서 구축한 실습 프로젝트입니다.
