# DNS 조회와 결과 해석

**조회 명령만 포함합니다.** 예제 주소를 실제 DNS 주소로 치환합니다. 테스트하는 내부·외부 회선과 조회 시각을 기록합니다.

## Linux: dig

```bash
# 특정 권한 서버의 원본 응답
dig @192.0.2.234 agri.example.org. A +norecurse
dig @192.0.2.234 example.org. SOA +norecurse
dig @192.0.2.234 example.org. NS +norecurse

# TCP 53도 별도로 확인
dig @192.0.2.234 agri.example.org. A +tcp +norecurse

# 이 PC가 평소 사용하는 DNS 경로
dig agri.example.org. A

# 상위부터 위임 경로 확인
dig +trace example.org. NS
```

| 요소 | 뜻 |
|---|---|
| `@IP` | 질문을 받을 DNS 서버 선택 |
| `+norecurse` | 재귀를 요청하지 않음. 권한 데이터 조회나 캐시 응답은 가능 |
| `+tcp` | TCP로 질의 |
| `+trace` | 위임 경로를 따라 조회. 네트워크 정책에 따라 중간 단계 실패 가능 |
| 이름 끝의 `.` | 검색 접미사가 붙지 않는 완전한 도메인 이름 |

## Windows: nslookup·포트 확인

```cmd
nslookup -debug agri.example.org. 192.0.2.234
nslookup -debug www.example.org. 192.0.2.234
nslookup -debug example.net. 192.0.2.234
nslookup agri.example.org.
```

마지막 명령은 기본 리졸버 경로입니다. `example.net`은 이 DNS가 권한을 갖지 않는다는 전제에서 재귀 제한 테스트에 사용합니다.

```powershell
Test-NetConnection 192.0.2.234 -Port 53
Test-NetConnection 192.0.2.228 -Port 1025
```

`Test-NetConnection -Port`는 **TCP** 검사입니다. DNS UDP 53 정상 여부를 증명하지 않습니다. TCP가 열려도 애플리케이션 응답·권한은 별도 검증합니다.

## 응답 읽기

| 출력 | 판단 |
|---|---|
| `NOERROR` + A 응답 | 해당 질의 성공 |
| `aa` 또는 `auth. answer` | 응답 서버가 해당 이름에 권한을 가지고 응답 |
| `ra` | 재귀 기능 가용성 표시. 이 클라이언트에 실제 재귀를 허용했다는 단독 증거 아님 |
| `rd` 또는 `want recursion` | 클라이언트가 재귀를 요청했다는 표시 |
| `NXDOMAIN` | 조회 이름이 없다는 응답 |
| `NOERROR`, AAAA 0개, SOA 포함 | 이름은 있지만 IPv6 레코드가 없는 NODATA일 수 있음 |
| `REFUSED` | 조회 정책 등으로 처리 거부 |
| `SERVFAIL` | 서버 처리 실패. 위임·DNSSEC·외부 질의·자원 등 원인 추가 확인 |
| timeout | 서버·포트·방화벽·경로·부하 확인 |
| 루트 NS 목록만 있고 answer 0개 | referral일 수 있음. 대상 A 레코드 조회 성공으로 해석하지 않음 |

### UnKnown과 .lan

nslookup은 처음에 DNS 서버 IP의 PTR을 조회할 수 있습니다. 이 PTR이 없어서 `서버: UnKnown`이 나와도 이후 A 조회가 정상이라면 그것만으로 DNS 장애가 아닙니다. 질문 항목을 확인하여 PTR의 NXDOMAIN과 실제 대상 도메인의 NXDOMAIN을 구분합니다.

`agri.example.org.lan`이 보이면 검색 접미사 조회입니다. 끝에 점을 붙여 재검사합니다. 이때 보이는 루트 캐시 referral만으로 외부 재귀가 성공했다고 단정하지 않습니다.

## 캐시와 반영 시간

TTL 86400은 24시간 뒤 활성화라는 뜻이 아닙니다. 원본 DNS는 reload 후 바로 응답할 수 있고, 재귀 DNS에는 기존 응답의 남은 TTL이 영향을 줍니다. 신규 이름도 이전 NXDOMAIN·NODATA 캐시가 남으면 지연될 수 있습니다. 음성 캐시는 SOA TTL과 SOA MINIMUM 등의 영향을 받습니다.

PC의 캐시를 비워도 외부 리졸버 캐시는 사라지지 않습니다. 따라서 무조건 '하루 후 해결'이라고 약속하지 않고 권한 DNS·기본 리졸버·다른 외부 회선의 결과를 분리합니다.

## 재귀 제한 검증 기준

허용 대상이 아닌 외부 클라이언트에서 원본 도메인은 계속 정상이어야 합니다. 외부 도메인은 최종 재귀·캐시 답변을 제공하지 않아야 합니다. 보통 REFUSED를 기대하지만 구형 패키지·설정별로 응답 형식이 다를 수 있습니다. 단일 고정 이름 한 번의 조회만으로 모든 ACL을 증명하지 않습니다. 기존 캐시 이름과 승인된 별도 테스트 이름을 비교하고, 서버 패킷·현재 view 설정도 대조합니다.

[접근 통제](04-access-control.md) · [위임 점검](06-delegation.md) · [RFC 및 도구 출처](../../REFERENCES.md)
