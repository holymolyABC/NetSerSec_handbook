# 신규 레코드 등록·변경·롤백

**대상:** 수동 관리하는 정적 master zone, `localhost_resolver` view. **영향:** 해당 도메인 이름 해석 변경. 백업·검사 후 zone 단위 reload.

## 1. 요청 내용 해석

요청 예: `agri.example.org`로 `http://192.0.2.228:1025/login`에 접속.

| 정보 | 처리 위치 |
|---|---|
| `agri.example.org → 192.0.2.228` | DNS A 레코드 |
| TCP 1025 접근 | 방화벽·서버 서비스 |
| `/login` 경로 | 웹 애플리케이션 |
| 주소창에서 포트 생략 | 기본 포트 80/443·역방향 프록시 |
| HTTPS | 인증서·TLS·웹서버 |

철자·IP·기존 레코드·사용할 프로토콜을 확인합니다. DNS에 URL·포트·경로를 넣지 않습니다. 대상 요청서의 경로에 공백이 있으면 담당자에게 확인합니다.

## 2. 현재 상태와 파일 조회

```bash
dig @192.0.2.234 agri.example.org. A +norecurse
dig @192.0.2.234 example.org. SOA +norecurse
grep -n 'agri' /var/named/chroot/var/named/example.org.zone
ls -l /var/named/chroot/var/named/example.org.zone*
```

기존 이름에 CNAME이 있으면 A 레코드를 중복 추가하지 않습니다. `.jnl`이 있거나 `allow-update`·`update-policy`가 설정됐다면 동적 업데이트 여부부터 확인합니다. 동적 zone은 운영 절차에 따라 `nsupdate` 또는 freeze/thaw 등을 별도 검토하며 이 정적 편집 절차를 중단합니다.

## 3. 백업과 편집 — 변경 단계

```bash
cp -a /var/named/chroot/var/named/example.org.zone /var/named/chroot/var/named/example.org.zone.bak-$(date +%Y%m%d-%H%M%S)
vi /var/named/chroot/var/named/example.org.zone
```

zone의 `$ORIGIN`이 `example.org.`인 구간에서 첫 칸부터 추가합니다.

```dns
agri    IN    A    192.0.2.228
```

현재 `$ORIGIN`이 다른 구간이면 완전한 이름을 사용합니다.

```dns
agri.example.org.    IN    A    192.0.2.228
```

행 앞 공백은 소유자 이름 생략으로 해석될 수 있으므로 레코드 이름 앞에 불필요한 들여쓰기를 넣지 않습니다.

### serial 증가

SOA serial을 현재 값보다 증가시킵니다. `YYYYMMDDNN`은 관리 관례이며 자동 계산 기능이 아닙니다. 예를 들어 기존 `2026041002`보다 큰 `2026101401`로 변경할 수 있습니다. 같은 날 두 번째 변경은 `2026101402`처럼 증가시킵니다. 날짜가 바뀌어도 현재 값보다 작게 되돌리지 않습니다.

### TTL

`$TTL 86400`을 상속하면 24시간입니다. 낮은 TTL이 필요한 경우 운영 정책에 따라 해당 레코드에 `agri 300 IN A ...`처럼 명시할 수 있습니다. 기존 캐시는 새 TTL로 소급 변경되지 않습니다. 등록 전 NXDOMAIN을 받은 리졸버는 음성 캐시가 남을 수 있습니다.

## 4. 검사 — 오류가 있으면 적용 중단

```bash
named-checkzone example.org /var/named/chroot/var/named/example.org.zone
```

`OK`와 새 serial을 확인합니다. 설정 파일도 변경했다면 별도로 `named-checkconf -t /var/named/chroot /etc/named.caching-nameserver.conf`를 수행합니다.

## 5. 적용

```bash
rndc reload example.org IN localhost_resolver
```

`zone reload queued`는 요청이 큐에 등록됐다는 뜻입니다. 실제 적용 여부는 다음 직접 조회와 로그로 확인합니다. zone 내용만 바꾸고 `rndc reconfig`만 실행하는 것으로 대체하지 않습니다.

## 6. 검증

```bash
dig @192.0.2.234 agri.example.org. A +norecurse
dig @192.0.2.234 example.org. SOA +norecurse
tail -n 30 /var/log/messages
```

정상 기준: `NOERROR`, `aa`, A 응답 `192.0.2.228`, 기대 serial. 외부 회선에서도 조회하고 평소 사용하는 재귀 DNS의 응답을 별도로 확인합니다. [조회 문서](03-query-and-validation.md)를 참고합니다.

웹 접속 검증은 브라우저 또는 승인된 클라이언트에서 별도로 진행합니다. DNS 응답만으로 TCP 1025·인증·로그인 성공을 보고하지 않습니다.

## 7. 롤백

백업 경로를 정확히 선택하여 원래 내용으로 복원하되 **serial은 마지막으로 배포한 값보다 더 크게 수정**합니다. 이 원칙은 보조 DNS와의 동기화를 위해 중요합니다.

```bash
# 아래 백업 이름은 예시이며 실제 생성한 파일을 사용
cp -a /var/named/chroot/var/named/example.org.zone.bak-20261014-130000 /var/named/chroot/var/named/example.org.zone
vi /var/named/chroot/var/named/example.org.zone
named-checkzone example.org /var/named/chroot/var/named/example.org.zone
```

검사 통과 후 같은 zone reload를 실행하고 A·SOA를 재조회합니다. 이미 캐시된 잘못된 응답은 원래 TTL 동안 남을 수 있습니다. 롤백 기록에 이 제한을 남깁니다.

## 기타 레코드

| 종류 | 예시 | 주의 |
|---|---|---|
| A | `agri IN A 192.0.2.228` | IPv4 주소만 등록 |
| AAAA | IPv6 주소 등록 | 실제 IPv6 서비스·경로 확인 |
| CNAME | `portal IN CNAME agri.example.org.` | 대상 이름 끝의 점, 같은 이름의 다른 데이터와 충돌 주의 |
| MX | `@ IN MX 10 mail.example.org.` | 메일 서버 이름과 주소 확인 |
| PTR | 역방향 zone에 등록 | 공인 IP 제공자나 역방향 관리 주체가 별도일 수 있음 |

[전체 zone 예시](../../examples/bind/example.org.zone.example) · [출처](../../REFERENCES.md)
