# 재귀·캐시·영역 전송 접근 통제

**수준:** 설정 예시. 사례의 최종 외부 검증 완료를 의미하지 않습니다. **대상:** 권한 DNS와 제한된 재귀 기능이 함께 있는 기존 구성.

## 역할 구분

| 기능 | 설정 | 운영 목적 |
|---|---|---|
| 일반 질의 허용 | `allow-query` | 관리 도메인의 권한 데이터도 이 제한을 받음 |
| 캐시 접근 | `allow-query-cache` | 다른 도메인의 저장된 응답 접근 제한 |
| 재귀 처리 허용 | `allow-recursion` | 다른 도메인을 대신 찾는 클라이언트 제한 |
| 재귀 기능 | `recursion yes/no` | 해당 options/view의 기능 활성화 여부 |
| 영역 전송 | `allow-transfer` | 보조 DNS 등에 zone 전체를 전달할 대상 제한 |

기존 `allow-query-cache { any; };`, `match-clients { any; };`, `recursion yes;` 조합과 무관한 외부 도메인 응답은 공개 리졸버 문제를 강하게 뒷받침합니다. 정책을 닫아도 외부 패킷 도착은 계속될 수 있습니다.

## 1. 변경 전 확인

전체 main/include 설정, view 순서와 override를 확인합니다. PC·메일·백업·모니터링 서버가 이 DNS를 재귀용으로 사용하는지, NAT 후 서버에 보이는 출발지가 무엇인지 조사합니다. 로그와 현재 응답을 확보하고 설정을 백업합니다.

```bash
cp -a /var/named/chroot/etc/named.caching-nameserver.conf /var/named/chroot/etc/named.caching-nameserver.conf.bak-$(date +%Y%m%d-%H%M%S)
```

## 2. 신뢰 대상만 허용하는 예시

**[주의: 서비스 영향 가능]** 기존 외부·내부 클라이언트의 인터넷 이름 해석이 중단될 수 있습니다. 실제 사용 IP를 조사한 뒤 목록을 정합니다.

```conf
// 최상위: options/view 밖. 기존 동일 이름 ACL 유무 확인
acl "trusted_resolvers" {
    127.0.0.1;
    ::1;
    // 필요한 경우 실제 내부 출발지 IP/대역을 검토해 추가
};
```

기존 view에 아래 항목을 병합하는 예입니다. 기존 zone include는 유지합니다.

```conf
view localhost_resolver {
    match-clients { any; };
    match-destinations { any; };
    allow-query { any; };
    allow-query-cache { trusted_resolvers; };
    allow-recursion { trusted_resolvers; };
    recursion yes;
    include "/etc/named.rfc1912.zones";
};
```

이 예시의 ACL은 loopback만 허용합니다. BIND 내장 ACL `localhost`는 단순히 `127.0.0.1`만 뜻하지 않고 서버 인터페이스 주소들도 포함할 수 있습니다. 엄밀한 범위가 필요하면 위처럼 명시합니다. options/view/zone 중 어디에 제한이 있는지와 상속을 확인합니다.

권한 전용으로 운영하고 서버 자체를 포함한 재귀 사용자가 없다면 `recursion no;`도 선택지입니다. 기존 재귀 의존성을 확인하지 않은 채 적용하지 않습니다. 캐시 접근 제한도 별도로 검토합니다.

## 3. 영역 전송

보조 DNS가 없다면 해당 options/view 또는 zone에 다음을 검토합니다.

```conf
allow-transfer { none; };
```

보조 DNS가 있다면 승인된 전송 대상과 인증 방식에 맞춰 허용합니다. `192.0.2.234/28`은 단일 호스트가 아니라 `.224~.239` 주소 범위입니다. 단일 IP 허용과 혼동하지 않습니다. 구문 검사에서 비정규 CIDR 표기가 거부되거나 경고되는지도 확인합니다.

TCP 53 전체를 막아 영역 전송을 통제하려 하지 않습니다. 일반 DNS도 TCP가 필요합니다.

## 4. 검사·적용·검증

```bash
named-checkconf -t /var/named/chroot /etc/named.caching-nameserver.conf
```

검사 성공한 경우에만:

```bash
rndc reconfig
```

권한 도메인 A·SOA, 내부 재귀 사용자, 허용되지 않은 외부 재귀·캐시 조회를 각각 검사합니다. `aa`·실제 응답값을 확인합니다. `ra` 플래그나 거부 로그 몇 줄만으로 전체 성공을 선언하지 않습니다.

권한 이름이 REFUSED라면 재귀 제한을 다시 `any`로 풀기 전에 실제 질문 이름·view·권한 zone 로드·A/CNAME 응답·현재 ACL을 확인합니다. 복구가 필요하면 사전 백업의 승인된 설정으로 되돌리고 검사 후 reconfig, 동일 검증을 수행합니다. 공개 캐시를 영구 대안으로 남기지 않습니다.

## 기타 보안 점검

기존 `query-source port 53;`은 외부 질의 출발지 포트를 고정하는 경고 원인입니다. 제거·변경 전 방화벽의 송신 및 응답 허용 정책을 확인합니다. 오래된 버전에 최신 RRL 설정을 그대로 붙여 넣지 않습니다. 레거시 운영 절차와 지원되는 OS/BIND로의 이전 계획을 분리합니다.

[로그와 방화벽 대응](05-logging.md) · [보안 분류](../security/README.md) · [버전별 ACL 공식 설명](../../REFERENCES.md)
