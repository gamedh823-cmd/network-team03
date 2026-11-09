# 동작 검증 결과 — 시나리오 3

> 팀 통합보고서(`최종_제출/ALEPH_3조_통합보고서.docx`, 4-3-8·4-3-9·4-3-11절)에 이미 기록된 실측 검증 결과를 발췌·정리한 문서입니다. 재현 자체는 이 통합보고서 작성 시점(11/3)에 이미 수행·확인되었습니다.

## 1. ESP32-C3 API 수동 검증

```powershell
Invoke-RestMethod -Uri "http://192.168.7.81/"

Invoke-RestMethod -Uri "http://192.168.7.81/warning" -Method Post `
  -Headers @{ "X-Auth-Token" = "<ESP32_AUTH_TOKEN>" }

Invoke-RestMethod -Uri "http://192.168.7.81/lockdown" -Method Post `
  -Headers @{ "X-Auth-Token" = "<ESP32_AUTH_TOKEN>" }

Invoke-RestMethod -Uri "http://192.168.7.81/unlock" -Method Post `
  -Headers @{ "X-Auth-Token" = "<ESP32_AUTH_TOKEN>" }
```

| 검증 항목 | 결과 |
| --- | --- |
| `GET /` 상태 조회 | 정상 — `device/status` 필드 반환 |
| `POST /warning` (정상 상태에서) | 정상 — 노랑 전환 확인 |
| `POST /warning` (Lockdown 상태에서) | 정상 — 무시됨 (하향 방지 로직 확인) |
| `POST /lockdown` | 정상 — 3회 깜빡임 후 빨강 고정 + 부저 경보 |
| `POST /unlock` | 정상 — 초록 복귀 |
| `POST /device/control` (Lockdown 중) | 정상 — 423 거부 확인 |

## 2. 자동 공격 재현 (킬체인 전 구간)

```powershell
cd scenario3-killchain/scripts
python test_killchain.py critical
```

- `recon_scan → api_key_compromised → device_control` 3단계를 동일 공격 IP(203.0.113.188)로 순차 전송
- Graylog 1분 실행 주기 내에 보드가 자동으로 빨강 + 경보로 전환되는 것을 확인 (사람 개입 없음)
- 공용 대시보드에 인시던트 티켓 **INC-KILLCHAIN-003** 자동 생성 확인
- n8n 실행 로그에서 CRITICAL 분기 전 노드(pfSense 차단 · ESP32 Lockdown · 티켓 생성 · Discord 알림) 전부 성공 확인

## 3. 오탐 방지(false positive) 검증

```powershell
python test_killchain.py warning
# recon_scan + api_key_compromised 2종만 전송 -> WARNING(노랑) 유지, CRITICAL로 격상되지 않음
```

이벤트 유형이 3종에 미달하면 CRITICAL로 격상되지 않고 WARNING(노랑)에 머무르는 것을 확인 — 완전한 킬체인(3단계)에 미달하는 공격 징후를 과도하게 CRITICAL로 올리지 않음을 검증.

## 4. 참고 — 재탐지 동작

Graylog의 10분 검색 윈도우와 5분 Grace Period 특성상, 수동으로 `/unlock`한 뒤에도 같은 공격 IP의 로그가 윈도우 내에 남아있으면 재탐지되어 다시 Lockdown으로 전환될 수 있습니다. 이는 공격 증거가 남아있는 한 방어 태세가 유지되는 의도된 동작입니다.

## 5. 시연 데모 시퀀스 (표 7)

| 순서 | 내용 |
| --- | --- |
| ① | 부팅 — 파랑. RST 버튼으로 재부팅 |
| ② | 정상 — 초록. Wi-Fi 연결 성공 시 자동 전환 |
| ③ | 상태 조회 — `GET /` |
| ④ | 경고 → Lockdown → 해제를 수동 호출로 하나씩 시연 |
| ⑤ | 디바이스 제어 호출 — 파랑 1회 깜빡임으로 평상시 명령 수신 확인 |
| ⑥ | 실제 공격 재현 — 경고용 IP로 2종 전송(WARNING 확인) → 새 IP로 3종 전송(CRITICAL 확인), pfSense 차단 표 0→1건, ESP32 빨강·Discord 알림·대시보드 티켓까지 전체 흐름 확인 |

## 비고

- incident_id 포맷·timestamp 필드가 공용 API 서버 검증 규칙과 정확히 일치하는지는 통합보고서 작성 시점에 공용 API 담당(김라연)과 교차 점검이 필요하다고 기록되어 있었습니다. 이후 실제 교차 점검 결과가 있다면 이 문서에 추가해주세요.
- 자세한 원본 서술은 `최종_제출/ALEPH_3조_통합보고서.docx`의 4-3-8 ~ 4-3-11절을 참고하세요.
