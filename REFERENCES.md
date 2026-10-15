# 출처와 근거

검토일: 2026-10-15. 실제 출력은 DNS 운영 대화를 기준으로 하고, 아래 문서는 동작 원리와 문법을 확인하는 참고 자료입니다. 최신 ARM의 모든 옵션이 레거시 패키지에서 지원된다는 뜻은 아닙니다.

| 자료 | 확인하는 내용 |
|---|---|
| [ISC: 재귀·캐시 ACL 동작 변경](https://kb.isc.org/docs/aa-00269) | 버전별 ACL 기본값 차이, 권한 데이터와 캐시 접근 구분 |
| [ISC: allow-query-cache와 allow-recursion](https://kb.isc.org/docs/aa-00503) | 두 접근 제한의 관계 |
| [ISC: BIND 로그 구성](https://kb.isc.org/v1/docs/aa-01526) | channel·category, 크기·보관 개수, debug 동작 |
| [BIND ARM 9.16.38: manpages](https://bind9.readthedocs.io/en/v9.16.38/manpages.html) | named-checkconf·named-checkzone·rndc 문법 비교 참고 |
| [BIND ARM: reference](https://bind9.readthedocs.io/en/v9.16.18/reference.html) | query-source 고정 포트와 무작위화 주의 |
| [ISC: Recursive best practices](https://kb.isc.org/docs/bind-best-practices-recursive) | 공개 재귀 방지, 메모리·캐시 운영 검토 |
| [ISC: Authoritative best practices](https://kb.isc.org/docs/bind-best-practices-authoritative) | 권한 DNS 운영과 재귀 분리 |
| [RFC 2308](https://www.rfc-editor.org/rfc/rfc2308.html) | NXDOMAIN·NODATA의 음성 캐시 TTL |
| [NGINX: HTTP proxy module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) | proxy_pass와 전달 헤더 |
| [NGINX: server names](https://nginx.org/en/docs/http/server_names.html) | Host 기반 가상 호스트 구분 |
| [NGINX: reverse proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/) | 역방향 프록시 구성 참고 |
| [IBM: IMM과 usb0](https://www.ibm.com/support/pages/usb0-down-red-hat-enterprise-linux-5x-after-imm-reset-ibm-system) | 내부 USB 관리망의 169.254.95.x 주소 |

## 근거 해석 원칙

- `dig @권한DNS` 성공은 그 DNS에 직접 물은 결과입니다. 상위 위임과 모든 공용 리졸버가 정상이라는 증거는 아닙니다.
- `DiG` 버전과 named 버전을 구분합니다. 레거시 Red Hat 패키지에는 기능이 백포트됐을 수 있습니다.
- 운영 대화에서 안내된 설정은 최종 설정 전체·검사·외부 검증이 없으면 적용 완료로 간주하지 않습니다.
- 문서 작성 과정의 정적 검사는 운영 서버의 구문 검사와 서비스 검증을 대체하지 않습니다.
