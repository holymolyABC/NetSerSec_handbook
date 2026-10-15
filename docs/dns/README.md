# DNS 운영

이 절은 레거시 chroot BIND 운영 사례를 바탕으로 합니다. 설치 자동화 대신 현재 구성을 확인하고 안전하게 변경하는 데 초점을 둡니다.

| 순서 | 문서 | 핵심 |
|---|---|---|
| 1 | [구성 확인](01-bind-configuration.md) | 프로세스 옵션·chroot·include·view·zone 경로 |
| 2 | [레코드 등록](02-record-management.md) | A·CNAME·serial·검사·reload·롤백 |
| 3 | [조회와 검증](03-query-and-validation.md) | dig·nslookup·TTL·오류 구분 |
| 4 | [접근 통제](04-access-control.md) | 권한 조회·재귀·캐시·영역 전송 |
| 5 | [로그 관리](05-logging.md) | 메시지 해석·파일 분리·순환·검증 |
| 6 | [위임 점검](06-delegation.md) | 등록기관·상위 NS·glue·하위 zone |
| 7 | [장애 진단](07-troubleshooting.md) | 부하·패킷·TCP/UDP·확인 순서 |

예제에서 사용하는 역할:

| 역할 | 예시 |
|---|---|
| 관리 도메인 | `example.org` |
| 권한 DNS | `ns.example.org` / `192.0.2.234` |
| 애플리케이션 서버 | `192.0.2.228` / HTTP 1025 |
| 신규 이름 | `agri.example.org` |
| 외부 조회용 테스트 이름 | `example.net` — 이 서버가 관리하지 않는다는 전제 |

운영 실주소가 아닙니다. 이전 작업의 성공 여부는 [사례 기록](../cases/2026-10-dns-operations.md)에서 확인합니다.
