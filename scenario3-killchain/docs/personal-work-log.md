# 개인 작업 내역 — 유지원 (팀장 · 시나리오 3 담당)

이 문서는 EngineSquare 프로젝트에서 제가 직접 설계·구현·검증한 범위를 팀 공통 작업과 구분해 정리한 것입니다.

## 팀 공통 vs 개인 작업 구분

| 구분 | 내용 | 비고 |
| --- | --- | --- |
| 팀 공통 | 공용 Flask API 서버(`app.py`), 공통 웹앱/대시보드(`web-app/`), MySQL 스키마, 전체 아키텍처 다이어그램 | 팀원 공동 설계, 저는 시나리오3 연동 부분(`/automation/killchain-response`)만 직접 작성 |
| 개인(유지원) | 시나리오 3 전체 — Graylog 상관분석 규칙, n8n SOAR 워크플로, ESP32-C3 펌웨어, pfSense 연동, 테스트/데모 스크립트 | 설계부터 구현·디버깅까지 전담 |
| 팀원(강아름) | 시나리오 1 — SOAR 로그 버퍼링 미들웨어 | 본인 작성, 저는 시나리오3 상관분석 쿼리에서 이벤트 필드 연동만 함께 확인 |
| 팀원(김라연) | 시나리오 2 — 유출 API 키 탐지·자동 폐기/재발급 | 본인 작성, 저는 전체 발표자료 취합 과정에서만 다룸 |

## 직접 작업·검증한 항목 (시나리오 3)

- **Graylog 상관분석 규칙 설계**: `client_ip`로 group-by, 10분 윈도우, `event_type` 카디널리티 조건(`card(event_type) >= 3` → CRITICAL, `1~2` → WARNING) 직접 정의. 배경 트래픽(`spike_test_demo.py`가 생성하는 `192.168.10.*` 대역)을 쿼리에서 제외하는 노이즈 필터 추가.
- **n8n SOAR 워크플로 구현**: Graylog 알림 수신 → CRITICAL/WARNING 분기 → pfSense 차단·ESP32 Lockdown·인시던트 티켓 생성·디스코드 알림 4갈래 병렬 실행 설계 및 구현.
- **ESP32-C3 펌웨어 작성**: MicroPython으로 독립 HTTP 서버 구현(`/`, `/warning`, `/lockdown`, `/unlock`, `/device/control`), LED 4색 상태 표시, `X-Auth-Token` 헤더 인증, Lockdown 중 장치 제어 요청 423 거부 로직.
- **pfSense 연동**: n8n SSH 노드로 방화벽 별칭에 공격자 IP 자동 등록/해제.
- **통합 테스트**: `test_killchain.py`(합성 이벤트), `demo_killchain_live.py`(실제 Nmap 스캔 + API 공격 + ESP32 호출까지 이어지는 엔드투엔드 데모) 작성 및 실행.
- **팀 통합**: 전체 아키텍처 다이어그램 정리, 시나리오1·2·3 통합 테스트 진행, 최종 발표자료·통합보고서 취합.

## 근거 자료

- 회의록(`3조 프로젝트/회의록/ALEPH_활동일지_*.docx`) — 날짜별 작업 내용
- n8n 워크플로 export(`scenario3-killchain/n8n-workflows/scenario3-killchain-response.json`) — 커밋 이력으로 수정 과정 확인 가능
- Graylog Content Pack(`scenario3-killchain/graylog/scenario3_killchain_content_pack.json`)
- 최종 발표자료(`최종_제출/ALEPH_3조_발표자료_최종.pptx`) 슬라이드 27~45 — 시연·문제해결 캡처 포함

## 재현·검증 기록 링크

- 단계별 명령·기대 결과·실측 결과·캡처: [`reproduction-guide.md`](reproduction-guide.md), [`verification-results.md`](verification-results.md), [`images/`](images/)
- 새 PC 설치 절차와 "팀원 PC에만 있던 설정" 제거 내역: [`../../../docs/SETUP-FROM-SCRATCH.md`](../../../docs/SETUP-FROM-SCRATCH.md) 12절
- 커밋 이력: https://github.com/gamedh823-cmd/network-team03/commits/main (팀 원본: https://github.com/altdmfk/network-team03/commits/main)
- 팀원 코드 출처: 시나리오 1(강아름) `scenario1-middleware/`, 시나리오 2(김라연) `scenario2-api-key-compromised/`, 공용 API·웹앱은 팀 공동 — 본 저장소에서 개인 수정은 민감정보 제거(자리표시자화)와 DB 비밀번호 기본값 제거뿐입니다.
