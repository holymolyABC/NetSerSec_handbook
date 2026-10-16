# NetSerSec Handbook — 인프라 운영 핸드북

서버·네트워크·보안 엔지니어의 작업을 다시 실행할 수 있는 절차와 근거로 남기는 한국어 운영 매뉴얼입니다.

**저장소:** [holymolyABC/NetSerSec_handbook](https://github.com/holymolyABC/NetSerSec_handbook) · 공개 문서에는 익명화한 예제만 기록합니다.

**문서 기준일:** 2026-10-16 · **초판:** v0.1 · **형식:** GitHub Markdown

## 빠른 찾기

| 하고 싶은 작업 | 문서 |
|---|---|
| DNS 구성과 실제 설정 파일 찾기 | [BIND 구성 확인](docs/dns/01-bind-configuration.md) |
| 신규 도메인·A 레코드 추가 | [레코드 등록과 롤백](docs/dns/02-record-management.md) |
| dig·nslookup 결과 읽기 | [조회와 검증](docs/dns/03-query-and-validation.md) |
| 외부 재귀·캐시 조회 제한 | [재귀와 영역 전송 제한](docs/dns/04-access-control.md) |
| named.run·messages 로그 관리 | [로그 해석과 순환 저장](docs/dns/05-logging.md) |
| 상위 NS와 실제 DNS 목록 비교 | [위임과 네임서버 점검](docs/dns/06-delegation.md) |
| DNS 장애·부하·패킷 확인 | [DNS 장애 진단](docs/dns/07-troubleshooting.md) |
| 주소창에서 :1025 없애기 | [역방향 프록시](docs/server/web/reverse-proxy.md) |
| SCP 백업 중 수신 파일 크기 실시간 확인 | [백업 진행 상태 확인](docs/server/backup/scp-progress-monitoring.md) |
| 구형 Linux 시간 확인·보정 | [시간 점검](docs/server/linux/time-check.md) |
| 실제 작업에서 무엇이 확인됐는지 보기 | [익명화한 DNS 운영 사례](docs/cases/2026-10-dns-operations.md) |
| 새 매뉴얼·작업·장애 기록 만들기 | [공통 양식](templates/README.md) |
| 새 GitHub 저장소에 올리기 | [GitHub 등록과 운영](docs/governance/github-workflow.md) |

## 카테고리

| 분류 | 범위 | 현재 상태 |
|---|---|---|
| [DNS](docs/dns/README.md) | BIND, 레코드, 조회, ACL, 로그, 위임 | 상세 매뉴얼 작성 |
| [서버](docs/server/README.md) | Linux·Windows, 웹/WAS, 시간, 스토리지·백업 | 시간·웹 프록시·백업 진행 확인 작성 |
| [네트워크](docs/network/README.md) | 스위치, 라우팅, 무선, VLAN, 접속 경로 | 분류와 작성 기준 마련 |
| [보안](docs/security/README.md) | 방화벽, VPN, 취약점 조치, 접근 통제 | DNS 트래픽 대응 기준 작성 |
| [작업 사례](docs/cases/2026-10-dns-operations.md) | 실제 증거·변경·검증·미완료 사항 | DNS 사례 작성 |
| [문서 관리](CONTRIBUTING.md) | 검증 수준, 파일명, 출처, 변경 이력 | 작성 |

## 사용 전 읽기

- 모든 예제 도메인은 `example.org`, 공인 주소 예시는 `192.0.2.0/24`·`198.51.100.0/24`·`203.0.113.0/24` 문서용 대역으로 치환했습니다. 실제 운영 주소로 바꿔야 합니다.
- **레거시 운영 사례**는 BIND 9.3.6 기반 Red Hat 패키지와 chroot 구성입니다. 신규 서버 구축 권장안이나 최신 버전 설치 안내가 아닙니다.
- `DiG 9.3.6...` 출력은 조회 도구의 버전입니다. 실행 중인 named의 정확한 버전은 별도 확인해야 합니다.
- 명령은 **조회 → 백업 → 변경 → 검사 → 적용 → 검증 → 롤백** 순서로 사용합니다. 명령 묶음을 한 번에 붙여 넣지 않습니다.
- DNS 정상과 웹사이트 정상은 다릅니다. 웹서비스 포트·방화벽·TLS·애플리케이션은 별도로 검증합니다.
- 예시 파일은 운영 파일 전체를 대체하지 않습니다. 기존 include, view, controls, 로그 분류를 먼저 확인합니다.
- 최신 문서에 있는 옵션이 구형 패키지에서 지원된다고 가정하지 않습니다. 대상 서버의 `man`과 구문 검사를 우선합니다.

## 증거 수준

| 표기 | 뜻 |
|---|---|
| 확인됨 | 대화에 실제 명령 출력 또는 설정이 제시됨 |
| 예시 | 적용 방법을 설명하는 구성. 운영 적용 완료를 의미하지 않음 |
| 추정 | 정황으로 제시한 원인 후보 |
| 확인 필요 | 최종 설정·외부 검증 등 자료가 없음 |

초판은 DNS 운영 대화에서 재구성했습니다. 그 외 장비의 실제 작업이 완료됐다고 기록하지 않았습니다. [확인 범위와 정정 사항](docs/cases/2026-10-dns-operations.md)을 함께 읽으세요.

## 관리 원칙

문서는 작업 종류로 분류하고, 실제 작업 이력은 사례 문서에서 연결합니다. 비밀번호·키·인증 토큰·고객 연락처·원본 설정 덤프는 저장하지 않습니다. 실제 환경 정보가 필요한 운영대장은 접근 통제된 별도 장소에서 관리합니다.

[문서 작성 규칙](CONTRIBUTING.md) · [출처](REFERENCES.md) · [변경 이력](CHANGELOG.md) · [초판 검증 범위](VALIDATION.md)
