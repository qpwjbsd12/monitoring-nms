# 대학 IT 인프라 통합 모니터링(NMS) 구축

> 서버 16대 · 네트워크 장비 10대의 상태를 **Prometheus · Zabbix**로 집단 수집하고
> **Grafana** 통합 대시보드 한 화면에서 장애와 이상 징후를 실시간 관제한 프로젝트입니다.
> (메가스터디 AI 캠퍼스 정보보안 전문가 과정 3차 파이널 / 2026.09 ~ 2026.10)

![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?logo=prometheus&logoColor=white)
![Zabbix](https://img.shields.io/badge/Zabbix-CC0000?logo=zabbix&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?logo=grafana&logoColor=white)
![Ansible](https://img.shields.io/badge/Ansible-EE0000?logo=ansible&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?logo=linux&logoColor=black)

---

## 담당 역할
4개 팀(네트워크·서버·레드·퍼플) 협업 프로젝트에서 **네트워크 설계**와 **통합 모니터링(NMS) 구축**을 담당했습니다. 본 저장소는 그중 **모니터링** 부분을 정리한 것입니다.

---

## 1. 무엇을 만들었나

- **관제 목적**: 서비스 가용성 확보, 자원 이상·장애 조기 탐지, 공격으로 인한 영향 범위 확인
- **관제 범위**: 관제망·전산망·DMZ 서버, Area 0·DMZ 라우터 및 L3 스위치
- **한 화면 통합**: Prometheus·Zabbix 수집 데이터를 Grafana 한 곳에 모아 판단

---

## 2. 수집 구조 — 대상에 따라 도구를 나눈 이유

| 대상 | 수집 도구 | 수집 방식 | 이유 |
|---|---|---|---|
| 서버 Metric | **Prometheus** | Exporter Pull (HTTP 스크레이프) | PromQL 기반 알람 룰 적용에 적합 |
| **NAT 뒤 서버** | **Zabbix** | Zabbix Agent **Active 모드** | 아래 ★ |
| 네트워크 장비 | **Zabbix** | ICMP 폴링 + SNMP 폴링 | 장비엔 에이전트 설치 불가 |
| 시각화·통합 | **Grafana** | Prometheus·Zabbix 데이터소스 연동 | 한 화면 관제 |

> ★ **핵심 설계 포인트**: NAT 구간 서버에 Zabbix Agent **Passive**(인바운드)로 설정하면 포트포워딩으로 10051 포트를 열어야 해서 **공격 표면이 늘어납니다.** 그래서 **Active 모드(아웃바운드)**로 구성해, 인바운드 포트를 열지 않고도 NAT 내부 서버를 수집하도록 설계했습니다.

---

## 3. 대시보드 구성 (Grafana)

| 패널 그룹 | 표시 항목 |
|---|---|
| 종합 현황 | Prometheus 대상 수, Zabbix 관제 호스트 수, 현재 발생 알람 건수 |
| 장애·알람 | Prometheus 알람(등급·대상·구역), Zabbix 문제 목록(Severity·Host Group) |
| 서버 자원 | 서버별 CPU·메모리·디스크 사용률, 네트워크 송수신 트래픽 |
| NAT 구간 서버 | 에이전트 연결 상태, CPU·메모리·디스크 |
| 네트워크 장비 | 장비 ICMP UP/DOWN, 응답 시간, 패킷 손실, 장비 CPU, 인터페이스 트래픽 Top 10 |

<!-- TODO: docs/dashboard-overview.png (종합 현황 화면) -->
<!-- TODO: docs/network-devices.png (네트워크 장비 UP/DOWN 패널) -->

---

## 4. 알람 규칙 (탐지 → 대응)

| 탐지 항목 | 탐지 방법 | 기준 | 대응 |
|---|---|---|---|
| 서버 수집 불가(HostDown) | Prometheus `up` | `up == 0` | 서버·Exporter 프로세스 확인, 동일 구역 다수 시 네트워크 장애·공격 점검 |
| 서버 CPU 과다 | Prometheus 알람 룰 | 80% 이상 | 상위 프로세스 확인, 비인가 프로세스(마이닝·웹셸) 점검 |
| 서버 트래픽 이상 | Prometheus 알람 룰 | 평시 대비 배수 초과 | 출발지·목적지 확인, IDS/WAF 로그와 대조해 DoS·유출 여부 판단 |
| NAT 서버 에이전트 단절 | Zabbix 에이전트 가용성 | 연결 끊김 | 서버 상태·10051/TCP 경로 확인 |
| 디스크 I/O 과다 | Zabbix 트리거 | read/write 임계 초과 | I/O 유발 프로세스, 대량 파일 쓰기·암호화(랜섬웨어) 점검 |
| 네트워크 장비 다운 | Zabbix ICMP 폴링 | 응답 없음, 패킷 손실 100% | 장비 콘솔·라우팅·이중화 경로 확인 |

---

## 5. 공격 탐지 사례 (레드팀 UDP Flood)

레드팀 시나리오에서 전산망 라우팅 대상 **UDP Flood** 공격이 발생했을 때:

- 네트워크 장비 **MA0-R2 ~ R8 (7대)가 DOWN**으로 전환, ICMP 패킷 손실 **100%** 확인
- **MA0-R1, MA4-R1, MA4-L3SW1은 UP 유지**
- → 영향 범위를 **MA0-R2~R8 구간으로 즉시 특정**

<!-- TODO: docs/attack-before.png (공격 전 전부 UP) -->
<!-- TODO: docs/attack-after.png (공격 후 MA0-R2~R8 DOWN) -->

단순 알람 확인에서 끝내지 않고, **IDS/WAF 탐지 시점과 자원·가용성 지표 변화를 교차 대조해 "일반 장애 vs 공격 피해"를 판단**하는 흐름까지 구성했습니다.

---

## 6. 운영 자동화
- **Ansible**로 장비·서버 설정 및 수집 에이전트 배포 자동화
- NTP(chrony)로 전 서버 시각 동기화 → 로그·증적 시각 정렬

---

## 7. 트러블슈팅 — 트래픽 부하로 구조 변경
초기엔 가용성을 위해 관제 서버·회선을 이중화했으나, 운영 중 수집 트래픽이 집중되며 부하가 발생했습니다. 부하 원인을 분석해 **일부 이중화를 제거하고 관제 서버 네트워크 구조를 변경**, 수집 경로를 단순화해 안정적인 운영 구조를 확보했습니다.
→ 설계안이 실측에서 그대로 동작하지 않을 수 있으며, **데이터를 근거로 구조를 조정**하는 경험을 얻었습니다.

---

## 8. 사용 기술
`Prometheus` `Zabbix` `Grafana` `Alertmanager` `node_exporter` `SNMP` `Ansible` `Linux(Rocky/Ubuntu)` `OSPF` `VLAN` `NAT`

---

> 담당: **강병진** · 정보보안 전문가 과정 수료 예정(2026.10)
