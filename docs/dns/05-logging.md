# BIND 로그 해석·분리·순환 저장

**대상:** `/var/log/messages`와 chroot 내부 `data/named.run`을 사용하는 사례. **수준:** 로그 해석은 실제 출력 기반이며 새 logging 구성은 적용 확인이 없는 예시입니다.

## 1. 용량과 최근 로그 조회

```bash
df -h
du -sh /var/log /var/named/chroot/var/named/data
ls -lh /var/log/messages* /var/named/chroot/var/named/data/named.run*
tail -n 50 /var/named/chroot/var/named/data/named.run
tail -n 50 /var/log/messages
rndc status
```

큰 파일 전체를 cat으로 출력하지 않습니다. 파일 크기와 파일시스템 여유 공간을 구분합니다. 사례의 `/var`는 사용률 20%, 여유 37GB로 즉시 고갈 상태는 아니었습니다.

## 2. 메시지 해석

| 메시지 | 의미 | 판단 |
|---|---|---|
| `query (cache) '.../A/IN' denied` | 해당 클라이언트의 캐시 조회 거부 | 해당 요청 차단. 모든 외부 재귀 차단 완료의 증거는 아님 |
| `unexpected RCODE (REFUSED) resolving ...: IP#53` | 우리 서버가 질의하던 중 상대 DNS에서 거부 | 상대 IP는 최초 요청자의 IP가 아님 |
| `bad zone transfer request ... non-authoritative zone (NOTAUTH)` | 관리하지 않는 도메인의 영역 전송 거부 | 전송 성공이나 정보 유출을 의미하지 않음 |
| `loading configuration ...` | 설정 읽기 시작 | 적용 완료와 서비스 정상은 추가 검증 |
| `using specific query-source port ... insecure` | 외부 질의 출발지 포트 고정 | 방화벽 의존성 확인 후 개선 검토 |
| `no IPv6 interfaces found` | IPv6 인터페이스 없음 | IPv4 장애의 단독 근거 아님 |
| `DHCPREQUEST → DHCPACK → bound` | DHCP 갱신 성공 | usb0는 내부 관리망일 수 있음 |

반복 도메인은 자동화·오용 가능성을 보여주지만 공격 의도나 주체를 확정하지 않습니다. UDP 출발지 IP는 위조될 수 있습니다. 서버 시간 보정 전후에는 로그 시각이 역전될 수 있습니다.

## 3. 저장 위치와 분류

`channel`은 저장 위치·상세 수준, `category`는 메시지 종류입니다. named.run의 크기 제한은 messages에 적용되지 않습니다. 분류가 불확실하면 시험 로그에 `print-category yes;`를 지정해 확인합니다. 기존 default 채널과 view 안의 로그 설정도 확인합니다.

## 4. 분리·순환 설정 예시 — 변경 단계

**[주의: 로그 보관에 영향]** 순환 시 오래된 기록이 삭제됩니다. 장애 증거를 먼저 안전한 위치에 보존하고 main 설정을 백업합니다. 기존 logging 블록에 병합하며 중복 블록을 만들지 않습니다.

```conf
logging {
    channel default_debug {
        file "data/named.run" versions 5 size 20m;
        severity dynamic;
    };
    channel security_log {
        file "data/security.log" versions 5 size 20m;
        severity info;
        print-time yes;
        print-category yes;
        print-severity yes;
    };
    channel transfer_log {
        file "data/transfer.log" versions 5 size 20m;
        severity info;
        print-time yes;
        print-category yes;
        print-severity yes;
    };
    category security { security_log; };
    category xfer-out { transfer_log; };
};
```

security와 xfer-out만 명시적 채널로 보냅니다. 다른 분류는 기존·기본 경로를 따릅니다. NOTAUTH 메시지의 실제 분류는 대상 패키지의 기록으로 확인합니다. 모든 경고를 없애려고 전체 로그를 null로 보내지 않습니다.

호스트 OS에서 보는 파일은 `/var/named/chroot/var/named/data/` 아래에 있습니다. named의 디렉터리 쓰기 권한을 확인합니다. SELinux가 원인이면 거부 기록과 라벨을 확인하고, 전체 비활성화나 chmod 777로 해결하지 않습니다.

## 5. 검사·적용·검증

```bash
named-checkconf -t /var/named/chroot /etc/named.caching-nameserver.conf
```

검사 성공 후에만:

```bash
rndc reconfig
```

```bash
ls -lh /var/named/chroot/var/named/data/*log* /var/named/chroot/var/named/data/named.run*
tail -n 20 /var/named/chroot/var/named/data/security.log
tail -n 20 /var/named/chroot/var/named/data/transfer.log
tail -n 30 /var/log/messages
```

해당 이벤트가 발생해야 출력 여부를 확인할 수 있습니다. 파일이 없다면 이벤트 미발생·권한·설정 적용 실패·다른 분류를 구분합니다. A와 SOA 응답도 확인합니다. 문제 발생 시 백업 설정을 복원하고 검사 후 reconfig, 동일 검증을 수행합니다.

## 6. 크기와 보관의 한계

각 채널의 `versions 5 size 20m`은 현재 약20MB와 과거5개, 통상 약120MB 수준의 목표입니다. 기존 거대 파일과 쓰기 단위 때문에 일시적으로 초과할 수 있습니다. 기존6.3GB 파일은 잘리는 것이 아니라 순환 파일에 남을 수 있습니다. 모든 파일 합계의 제한이 아닙니다.

`severity dynamic`은 debug 수준에 따라 동작합니다. rndc status를 보고 불필요한 debug가 켜졌다면 증거를 보존한 후 `rndc notrace`를 검토합니다. 이 명령은 상태 변경이므로 자동 실행하지 않습니다.

messages는 syslog·logrotate 등 별도 관리입니다. `/etc/logrotate.conf`, `/etc/logrotate.d/syslog`와 실제 서비스를 확인합니다. 로그 이동은 과거 messages를 삭제하지 않습니다. 쓰기 중인 파일의 무작정 rm은 피합니다.

[구성 예시](../../examples/bind/logging.conf.example) · [공식 자료](../../REFERENCES.md)
