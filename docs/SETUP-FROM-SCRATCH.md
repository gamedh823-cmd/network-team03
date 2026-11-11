# 처음부터 설치·실행 가이드 (새 PC / VM 기준)

> 목표: 이 문서 순서대로만 따라 하면 팀원 PC 설정 없이 EngineSquare 전체(웹·DB·Graylog·n8n·Wazuh·pfSense·ESP32)를 띄울 수 있게 한다.
> 표시: ✅ = 이 저장소 파일/명령으로 검증 가능, 🔧 = 환경(네트워크·장비)에 따라 값이 달라지는 수동 설정.
> 아래 Graylog/n8n/Wazuh/pfSense의 *설치 명령*은 각 제품 공식 표준 절차이며, 저장소에 들어 있는 설정 파일(콘텐츠 팩·워크플로 JSON)을 불러오는 부분이 이 프로젝트 고유 부분입니다.

## 0. 포트·주소 한눈에 보기

| 구성요소 | 주소 / 포트 | 비고 |
| --- | --- | --- |
| 공용 Flask API 서버 | `http://127.0.0.1:5000` | `python app.py` |
| 공통 웹앱·대시보드 | `http://127.0.0.1:5001` (`/dashboard`) | `python web-app/app.py` |
| MySQL | `127.0.0.1:3306` | 컨테이너 `mysql-server`, DB `enginesquare` |
| 시나리오1 미들웨어 | `:8000`(수집) / `:9100`(메트릭) | `scenario1-middleware/` |
| Prometheus / Grafana | `:9090` / `:3000` | 선택(observability 프로필) |
| Graylog | `http://127.0.0.1:9000` (웹), `:12201` (GELF HTTP) | |
| n8n | `http://127.0.0.1:5678` | 웹훅 `/webhook/killchain-alert` 등 |
| Wazuh dashboard | `https://127.0.0.1:443` | 선택(시나리오2 병행 분석) |
| pfSense | 관리 웹 + SSH(22) | 별칭 `EasyRuleBlockHostsWAN` |
| ESP32-C3 보드 | `http://<ESP32_IP>/` (80) | 시리얼 로그로 IP 확인 |

## 1. 공통 준비물

- Windows 10/11 + PowerShell (통합 실행 스크립트가 `.ps1`) — Linux/macOS는 4절의 개별 실행 명령 사용
- Docker Desktop(또는 Docker Engine + compose), Python 3.10+, Git
- (시나리오3 실제 하드웨어) ESP32-C3 SuperMini + LED 4색 + 저항 / Thonny(MicroPython 업로드)
- (실제 차단) pfSense VM 1대 — 없으면 pfSense 노드는 비활성화하고 나머지만 시연 가능(워크플로에서 `onError: continueRegularOutput` 설정)

```powershell
git clone https://github.com/gamedh823-cmd/network-team03.git
cd network-team03
Copy-Item .env.example .env      # 값 채우기(아래 2절). .env 는 .gitignore 대상
python -m pip install -r requirements.txt -r web-app/requirements.txt
```

## 2. `.env` 채우기

| 키 | 값 | 설명 |
| --- | --- | --- |
| `MIDDLEWARE_INGEST_URL` | `http://localhost:8000/ingest` | 시나리오1 미들웨어 수집 주소 |
| `MIDDLEWARE_API_KEY` | 미들웨어 `.env`의 수집 키와 동일 값 | 비어 있으면 인증 헤더 미전송 |
| `ENGINESQUARE_DB_*` | 3절 MySQL 컨테이너 설정과 동일 | 비밀번호는 **직접 정해서** 입력(코드 기본값 없음) |
| `ENGINESQUARE_KEY_DELIVERY_SECRET` | (선택) Fernet 키 | API·웹앱을 서로 다른 PC에서 돌릴 때만 |

## 3. MySQL (공용 DB) ✅

```powershell
docker run -d --name mysql-server -p 3306:3306 `
  -e MYSQL_ROOT_PASSWORD=<.env의 ENGINESQUARE_DB_PASSWORD와 동일> `
  -e MYSQL_DATABASE=enginesquare mysql:8
```

웹앱·API 첫 실행 시 스키마·관리자 계정을 자동 생성합니다(`web-app/app.py`, `security_store.py`).

## 4. 공용 API 서버 + 웹앱 ✅

```powershell
.\start-enginesquare.ps1          # :5000, :5001, Wazuh 수집기 동시 기동 (Enter로 종료)
# 또는 개별 실행
python app.py                     # 터미널 1
python web-app/app.py             # 터미널 2
```

확인: `curl http://127.0.0.1:5000/health` → `{"status": "ok"}`, 브라우저 `http://127.0.0.1:5001`.

## 5. Graylog 6.1 🔧

1. MongoDB + Graylog Data Node(또는 OpenSearch) + Graylog 6.1을 공식 Docker Compose로 기동 → `http://127.0.0.1:9000` 로그인(초기 비밀번호는 Graylog 설치 시 직접 설정한 값).
2. **System → Inputs**: `GELF HTTP` 입력 생성, 포트 `12201`.
3. **System → Configurations → URL Whitelist**: `http://aleph-n8n:5678/webhook/` 접두사(또는 실제 n8n 주소) 추가 — Graylog→n8n 웹훅 호출 허용. (콘텐츠 팩의 웹훅 주소가 `http://aleph-n8n:5678/webhook/killchain-alert`이므로 **Graylog와 n8n을 같은 Docker 네트워크에 두고 n8n 컨테이너 이름을 `aleph-n8n`으로** 하거나, 이벤트 알림의 URL을 본인 n8n 주소로 수정.)
4. **System → Content Packs → Upload** → [`scenario3-killchain/graylog/scenario3_killchain_content_pack.json`](../scenario3-killchain/graylog/scenario3_killchain_content_pack.json) → **Install**. (Killchain CRITICAL/WARNING 이벤트 정의 + 알림 포함)
5. 이벤트 정의 검색 쿼리가 `event_type:(recon_scan OR PORT_SCAN OR api_key_compromised OR BRUTE_FORCE OR device_control)`, `client_ip` 그룹핑, `card(event_type) >= 3`, 10분 윈도우인지 확인.
6. 시나리오2 규칙: [`scenario2-api-key-compromised/graylog/api_key_compromised_rule.json`](../scenario2-api-key-compromised/graylog/api_key_compromised_rule.json) 동일 방식으로 가져오기.

## 6. n8n 🔧

```powershell
docker network create aleph-net
docker run -d --name aleph-n8n --network aleph-net -p 5678:5678 -v n8n_data:/home/node/.n8n n8nio/n8n
```

1. `http://127.0.0.1:5678` 접속 → 계정 생성.
2. **Import from File** → `scenario3-killchain/n8n-workflows/scenario3-killchain-response.json` (시나리오1·2도 각 폴더의 JSON).
3. 워크플로 안의 **자리표시자**를 본인 환경 값으로 교체 (저장소에는 비밀·내부주소를 제거해 두었음):

   | 자리표시자 | 위치 | 넣을 값 |
   | --- | --- | --- |
   | `<ESP32_IP>` | `ESP32 Lockdown`, `ESP32 Warning` 노드 URL | 보드가 시리얼 콘솔에 출력한 IP |
   | `CHANGE_ME_SHARED_SECRET` | 두 ESP32 노드 `X-Auth-Token` | `config.py`의 `AUTH_TOKEN`과 동일 값 |
   | `<API_SERVER_IP>` | 티켓 생성 노드 URL | 공용 API 서버 PC IP (n8n이 Docker면 `host.docker.internal`) |
   | `<DISCORD_CHANNEL_ID>` + Bot credential | 디스코드 알림 노드 | 본인 디스코드 채널 ID, n8n Credentials에 봇 토큰 등록(토큰은 저장소에 넣지 않음) |
   | SSH credential | `pfSense IP 차단` 노드 | 7절에서 만든 Ed25519 개인키 |

4. 워크플로 **Active** 켜기 → 웹훅 URL `http://<n8n>:5678/webhook/killchain-alert` 가 Graylog 알림 URL과 일치하는지 확인.

## 7. pfSense 🔧

1. pfSense VM 설치, 관리 IP 설정. **System → Advanced → Admin Access**: SSH 활성화.
2. n8n 컨테이너 안에서 키 생성:
   ```bash
   docker exec -it aleph-n8n ssh-keygen -t ed25519 -f /home/node/.ssh/id_ed25519 -N ""
   docker exec aleph-n8n cat /home/node/.ssh/id_ed25519.pub
   ```
   공개키를 pfSense **System → User Manager → (계정) → Authorized SSH Keys** 에 등록, 개인키는 n8n SSH credential(Private Key)에 등록. *(비밀번호 인증은 n8n 내장 ssh2 협상 문제로 실패 → 개선점 문서 7번)*
3. 차단 명령은 워크플로가 `easyrule block wan <공격자IP>` 로 실행. 결과 확인: **Diagnostics → Tables → `EasyRuleBlockHostsWAN`**.

## 8. Wazuh 4.9 (선택, 시나리오2 병행 분석) 🔧

공식 single-node Docker 배포로 기동 후, 규칙 [`scenario2-api-key-compromised/wazuh/scenario2_rules.xml`](../scenario2-api-key-compromised/wazuh/scenario2_rules.xml)을 manager의 `/var/ossec/etc/rules/`에 복사하고 재시작. 로그 전달은 `start-enginesquare.ps1`이 `forward_api_logs.py`를 함께 실행. 자세한 절차는 시나리오2 README.

## 9. 시나리오1 미들웨어 ✅

```powershell
cd scenario1-middleware
Copy-Item .env.example .env
docker compose -f docker-compose.yml -f docker-compose.dev.yml --profile observability up -d --build
```

Graylog 없이 시험하려면 `docker-compose.mock.yml`(mock Graylog `:12201`) 사용. 상세: `SCENARIO1_SETUP_GUIDE.md`.

## 10. ESP32-C3 보드 ✅(펌웨어) / 🔧(배선)

1. 배선: [`scenario3-killchain/docs/hardware-wiring-guide.md`](../scenario3-killchain/docs/hardware-wiring-guide.md) (파랑 GPIO2 · 초록 GPIO7 · 노랑 GPIO10 · 빨강 GPIO0).
2. MicroPython 펌웨어를 보드에 설치 → `pico-firmware/config.py.example`을 `config.py`로 복사해 Wi-Fi·`AUTH_TOKEN` 입력 → `config.py`+`main.py`를 Thonny로 업로드.
3. 시리얼 콘솔 `Wi-Fi 연결됨: <ip>` 로 IP 확인 → 6절 `<ESP32_IP>`에 반영.

## 11. 설치 확인 체크리스트

```powershell
python -m unittest tests.test_incident_prefixes tests.test_killchain_response   # 7건 OK 기대
curl http://127.0.0.1:5000/health
python scenario3-killchain/scripts/test_killchain.py critical                    # 재현 절차는 reproduction-guide.md
```

## 12. "팀원 PC에만 있는 설정" 점검 결과 (이 문서 작성 시 처리한 것)

| 항목 | 이전 | 조치 |
| --- | --- | --- |
| MySQL 비밀번호 코드 기본값 | `web-app/app.py`, `security_store.py`에 하드코딩 | 기본값 제거, `.env`로만 주입 |
| 디스코드 채널 ID | n8n JSON 3개·시나리오1 가이드에 하드코딩 | `<DISCORD_CHANNEL_ID>`로 대체 |
| ESP32·API 서버 사설 IP | n8n JSON 하드코딩 | `<ESP32_IP>`, `<API_SERVER_IP>`로 대체 |
| ESP32 토큰 / Wi-Fi | `config.py`(gitignore), 워크플로는 `CHANGE_ME_SHARED_SECRET` | 템플릿(`config.py.example`)만 커밋 |
| n8n credential | credential ID·이름만 포함(비밀 없음) | 가져온 뒤 본인 credential 재연결 |
