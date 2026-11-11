# 시나리오 3 재현 가이드 — 공격 → 로그 수집 → 탐지 → n8n 자동 대응 → 복구

각 단계마다 **명령 / 기대 결과 / 실측 결과 / 근거**를 적었습니다. 실측 칸은 출처를 구분합니다.

- **[A] 11/3 통합 시연 실측** — 팀 통합보고서(4-3-8·4-3-9·4-3-11)에 기록된, 실제 Graylog·n8n·pfSense·ESP32 환경의 결과 (캡처: `images/`)
- **[B] 문서 작성 세션 재실행** — 이 저장소를 새 환경에 받아 그 자리에서 다시 돌린 결과 (하드웨어·SIEM 없이 가능한 범위)

> 시간·건수는 측정한 것만 적었고, 측정하지 않은 값은 비워 두었습니다(9절 "직접 채울 칸").

## 0. 사전 조건
[`../../docs/SETUP-FROM-SCRATCH.md`](../../docs/SETUP-FROM-SCRATCH.md) 5~10절 완료: Graylog GELF 입력(:12201) + 콘텐츠 팩, n8n 워크플로 Active, pfSense SSH, ESP32 온라인(초록).

## 1. 공격 + 로그 수집

```powershell
cd scenario3-killchain/scripts
$env:GRAYLOG_GELF_URL = "http://<Graylog PC IP>:12201/gelf"   # 같은 PC면 생략
python test_killchain.py warning     # recon_scan + api_key_compromised 2종
python test_killchain.py critical    # recon_scan + api_key_compromised + device_control 3종
# 실제 공격 버전(Nmap + API 키 재사용 + 장치 제어): python demo_killchain_live.py
```

| 항목 | 기대 | 실측 |
| --- | --- | --- |
| `warning` | 같은 `client_ip`로 이벤트 2건, 각 HTTP 202 | [B] 2건 전송, 전부 202, 약 0.06초 — 수신 로그에서 두 이벤트의 IP 동일 확인 |
| `critical` | 같은 `client_ip`로 이벤트 3건, 각 HTTP 202 | [B] 3건 전송, 전부 202, 약 0.06초 — `recon_scan`, `api_key_compromised`, `device_control` 동일 IP |
| 데이터 정합성(3단계 client_ip 일치) | 일치 | [A][B] 일치 |

> [B]는 Graylog 대신 로컬 GELF 수신 모의 서버(:12201)로 받아 *스크립트가 보내는 페이로드와 IP 일관성*만 확인한 것입니다. Graylog 집계는 [A] 기준.

## 2. 탐지 (Graylog 상관분석)

- 규칙: `client_ip` 그룹핑, 10분 윈도우, `card(event_type) >= 3` → CRITICAL / 2종 → WARNING (`graylog/scenario3_killchain_content_pack.json`)
- 확인 위치: Graylog **Alerts → Events**

| 입력 | 기대 | 실측 |
| --- | --- | --- |
| 2종 (`warning`) | WARNING, CRITICAL 아님(오탐 방지) | [A] 노랑 유지, CRITICAL 미격상 |
| 3종 (`critical`) | 1분 실행 주기 내 CRITICAL | [A] 사람 개입 없이 CRITICAL 발동 (캡처 `images/graylog_events.png`) |
| 재탐지 | 윈도우·Grace Period(5분)로 unlock 후 재 Lockdown 가능(의도) | [A] 확인 |

## 3. 자동 대응 (n8n)

Graylog → `POST /webhook/killchain-alert` → CRITICAL 분기: 티켓 생성 / pfSense 차단 / ESP32 Lockdown / 디스코드 알림. WARNING 분기: ESP32 `/warning` + 알림만.

| 대응 | 기대 | 실측 [A] | 근거 |
| --- | --- | --- | --- |
| n8n 실행 | CRITICAL 분기 전 노드 성공 | 전 노드 성공 | `images/n8n_workflow.png` |
| pfSense 차단 | `EasyRuleBlockHostsWAN` 0건 → 1건 | 0 → 1건 | `images/pfsense_block.png` |
| ESP32 | 3회 깜빡임 후 빨강 고정, `/device/control` 423 | 확인 | `images/board_critical.png` |
| 디스코드 | CRITICAL 임베드(공격 IP, 탐지 단계, 인시던트) | 확인 | `images/discord_critical.png` |
| 티켓 | `INC-KILLCHAIN-…` 생성 | `INC-KILLCHAIN-003` 확인 | 공용 대시보드 |

티켓 수신부(공용 API `POST /automation/killchain-response`)는 DB 없이 단위 테스트로 재검증:

```powershell
python -m unittest tests.test_incident_prefixes tests.test_killchain_response -v
```
→ 기대: 7건 OK. **실측 [B]: Ran 7 tests, OK (0.016s)** — CRITICAL 계약·한국시간 필드, 저장 실패 시 롤백, 키/차단 비변경, WARNING·잘못된 입력은 미기록, 접두사 규칙.

## 4. ESP32 API 수동 검증 [A]

```powershell
Invoke-RestMethod http://<ESP32_IP>/
Invoke-RestMethod http://<ESP32_IP>/warning  -Method Post -Headers @{ "X-Auth-Token" = "<AUTH_TOKEN>" }
Invoke-RestMethod http://<ESP32_IP>/lockdown -Method Post -Headers @{ "X-Auth-Token" = "<AUTH_TOKEN>" }
Invoke-RestMethod http://<ESP32_IP>/unlock   -Method Post -Headers @{ "X-Auth-Token" = "<AUTH_TOKEN>" }
```
결과표는 [`verification-results.md`](verification-results.md) 1절 (정상 6/6 항목).

## 5. 복구

Lockdown 해제는 자동이 아니라 **수동 승인**(팀 결정: 오탐으로 인한 서비스 중단 방지).

```powershell
Invoke-RestMethod http://<ESP32_IP>/unlock -Method Post -Headers @{ "X-Auth-Token" = "<AUTH_TOKEN>" }   # → 초록
# pfSense: Diagnostics → Tables → EasyRuleBlockHostsWAN 에서 해당 IP 삭제 (또는 쓰레기통 아이콘)
```
기대: LED 초록 복귀 / 차단 표에서 IP 제거. 실측 [A]: `/unlock` → 초록 복귀 확인. *주의: 공격 로그가 10분 윈도우에 남아 있으면 재탐지됨(2절).*

## 6. 시연 순서 요약 (발표 표 7)
①부팅 파랑 → ②정상 초록 → ③`GET /` → ④경고→Lockdown→해제 수동 → ⑤device 제어(파랑 1회) → ⑥실공격: 경고 IP 2종(노랑) → 새 IP 3종(CRITICAL, pfSense 0→1, 빨강, 디스코드, 티켓).

## 7. 코드 수정 포인트 (값을 바꾸거나 기능을 고칠 때)

| 바꾸고 싶은 것 | 위치 |
| --- | --- |
| CRITICAL 기준 종 수 / 윈도우 | Graylog 이벤트 정의 `card(event_type) >= 3`, Search Within 10분 (`graylog/…content_pack.json`) |
| 포함할 공격 유형 | 이벤트 정의 검색 쿼리의 `event_type:(…)` |
| 차단 방식 | n8n `pfSense IP 차단` 노드 명령 `easyrule block wan {{client_ip}}` |
| LED 색·핀 | `pico-firmware/config.py` (`LED_*_PIN`), 상태 로직은 `main.py` |
| Lockdown 중 경고 무시 / 423 | `pico-firmware/main.py` 의 `/warning`, `/device/control` 핸들러 |
| 티켓 필드·접두사 | `app.py` `/automation/killchain-response`, `tests/test_killchain_response.py` |

## 8. 알려진 한계
순서(정찰→침투→타격)는 미검증(3종 집합만 판정), Lockdown 상태는 보드 메모리라 재부팅 시 초기화, pfSense·보드 단일 장비. 개선 방향은 [`improvements-and-reflections.md`](improvements-and-reflections.md).

## 9. 직접 채울 칸 (현재 기록 없음)
| 측정 항목 | 방법 | 값 |
| --- | --- | --- |
| 정찰 시작 → Lockdown 완료 (s) | `critical` 실행 시각 ~ 보드 빨강 시각(스톱워치 또는 n8n 실행 시각) | |
| n8n 명령 → LED 점등 지연 (s) | n8n ESP32 노드 시작 ~ 점등 | |
| Graylog 집계 주기 대기 (≤60s) | Events 타임스탬프 − 마지막 로그 전송 시각 | |
