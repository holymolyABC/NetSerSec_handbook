# BIND 구성과 실제 설정 파일 찾기

**대상:** 레거시 Red Hat 계열·chroot BIND 사례. **수준:** 실제 실행 옵션·경로 확인을 바탕으로 한 절차. 신규 구축용 전체 구성 아님.

## 1. 실행 프로세스와 버전 조회

```bash
ps -ef | grep '[n]amed'
named -v
rpm -qa | egrep '^(bind|caching-nameserver)'
```

사례에서 확인된 실행 형식:

```text
/usr/sbin/named -u named -4 -c /etc/named.caching-nameserver.conf -t /var/named/chroot
```

| 옵션 | 뜻 |
|---|---|
| `-u named` | named 계정으로 실행 |
| `-4` | IPv4 사용 |
| `-c` | 설정 파일 지정 |
| `-t` | chroot 기준 경로 지정 |

명령 경로가 없으면 설치 위치·서비스 정의를 확인합니다. `systemctl`이 없는 구형 OS도 있으므로 최신 배포판 명령을 그대로 사용하지 않습니다.

## 2. 링크와 실제 파일 확인

```bash
ls -l /etc/named.conf /etc/named*.conf
cat /var/named/chroot/etc/named.caching-nameserver.conf
```

사례에서는 `/etc/named.conf`와 `/etc/named.caching-nameserver.conf`가 링크로 연결됐습니다. 파일명만 보고 실제 사용 파일을 단정하지 않습니다. 키가 포함된 설정을 외부로 공유하지 않습니다.

| chroot 내부에서 보이는 경로 | 호스트 OS에서 접근할 경로 |
|---|---|
| `/etc/named.caching-nameserver.conf` | `/var/named/chroot/etc/named.caching-nameserver.conf` |
| `/etc/named.rfc1912.zones` | `/var/named/chroot/etc/named.rfc1912.zones` |
| `/var/named/example.org.zone` | `/var/named/chroot/var/named/example.org.zone` |
| `/var/named/data/named.run` | `/var/named/chroot/var/named/data/named.run` |

## 3. include·view·zone 따라가기

```bash
grep -n 'include' /var/named/chroot/etc/named.caching-nameserver.conf
grep -n -A 15 -B 2 'example.org' /var/named/chroot/etc/named.rfc1912.zones
```

설정 해석 예:

```conf
// main config 일부
options {
    directory "/var/named";
};
view localhost_resolver {
    match-clients { any; };
    match-destinations { any; };
    recursion yes;
    include "/etc/named.rfc1912.zones";
};
```

```conf
// included zones 일부
zone "example.org" IN {
    type master;
    file "example.org.zone";
};
```

`file`이 상대 경로이면 `directory`를 기준으로 찾습니다. `view localhost_resolver`는 이름일 뿐입니다. `match-clients { any; };`이면 외부 클라이언트도 이 view에 들어올 수 있습니다. view가 여러 개라면 매칭 순서·목적지·동일 zone의 중복 정의를 확인합니다.

## 4. 검사 도구 구분

```bash
# 설정 문법 검사: 호스트에서 chroot를 지정
named-checkconf -t /var/named/chroot /etc/named.caching-nameserver.conf

# zone 파일 문법·일관성 검사
named-checkzone example.org /var/named/chroot/var/named/example.org.zone
```

첫 명령은 오류 없이 종료 코드 0, 두 번째는 `loaded serial ...`과 `OK`가 정상 기준입니다. `named-checkconf example.org ...`는 도메인을 파일로 해석하여 `file not found`가 발생할 수 있습니다. 구문 검사는 실제 응답 검증을 대체하지 않습니다.

## 작업 전 확인

- 실제 named 버전·패키지·OS를 기록했는가?
- symlink·chroot·directory·include를 따라 사용 중인 zone을 찾았는가?
- view별 ACL·forwarding·영역 전송 관계를 확인했는가?
- 동적 업데이트·`.jnl` 파일 사용 여부를 확인했는가? 동적 zone은 다음 문서의 정적 편집 절차를 그대로 적용하지 않습니다.

[다음: 레코드 등록](02-record-management.md) · [공식 문법 자료](../../REFERENCES.md)
