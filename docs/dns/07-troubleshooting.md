# DNS 장애·부하 진단

**초동은 조회부터 시작합니다.** 근거 없이 재시작, 캐시 전체 삭제, 외부53번 일괄 차단부터 하지 않습니다.

## 1. 증상별 우선 확인

| 증상 | 우선 확인 |
|---|---|
| 특정 이름만 실패 | 철자·A/CNAME·view·serial |
| 내부 또는 외부만 실패 | ACL·NAT 이후 출발지·UDP/TCP 경로 |
| 간헐 성공 | 상위 NS 목록과 각 DNS의 응답 |
| DNS 성공, 웹 실패 | 포트·HTTP Host·TLS·앱 |
| 대량 로그와 지연 | 증가 속도·메모리·swap·I/O·공개 재귀 |

## 2. 시스템 상태 조회

```bash
date '+%Y-%m-%d %H:%M:%S %Z %z'
uptime
free -m
top -b -n 1 | head -n 20
df -h
vmstat 1 5
```

사례에서는 named RES 약3.7GB, 시스템swap 약3.8GB, I/O wait8.6%였습니다. 메모리·I/O 압박 가능성은 있지만 단일 출력만으로 지연 원인이나 현재 swap 입출력을 단정하지 않습니다. vmstat의 si/so 변화와 DNS 응답 시간을 비교합니다.

## 3. 서비스와 설정 조회

```bash
ps -ef | grep '[n]amed'
netstat -lnptu | grep ':53'
rndc status
tail -n 50 /var/log/messages
```

구형 OS에는 ss·journalctl이 없을 수 있습니다. UDP/TCP와 대기 주소를 확인합니다. RNDC가 실패하면 controls·953번·인증 구성을 확인하며 키를 공유하지 않습니다. 구문 검사는 설정 파일과 zone을 구분해 수행합니다.

## 4. 짧은 패킷 캡처

```bash
tcpdump -pni any -s 0 'port 53' -c 50
```

전체 길이 캡처는 부담이 될 수 있으므로 건수를 제한하고 필요하면 `host 조사대상IP and port 53`으로 좁힙니다. PCAP에는 조회 도메인·IP가 포함되므로 별도 통제된 위치에서 보관합니다.

| 표시 | 해석 |
|---|---|
| 높은 출발지 포트 → DNS.domain | 질의 방향 후보. DNS 질문/응답 표시도 확인 |
| DNS.domain → 다른DNS.domain | 외부 질의일 수 있음. 내용과 앞뒤를 확인 |
| `[\|domain]` | 캡처 길이 부족 등으로 해석 내용이 잘림 |
| `packets dropped by kernel` | 캡처 과정의 누락. 실제 DNS 패킷 손실 수치가 아님 |

구형 query-source 고정 설정이 있으면 외부 질의도 출발지53번이므로 포트만 보고 방향을 확정하지 않습니다.

## 5. 조치 선택과 검증

설정 오류는 백업 후 최소 변경하고 구문 검사 후 적용합니다. 공개 재귀·대량 로그·FW 부하를 분리합니다. 재시작이 필요한 경우 로그·시각·프로세스 상태부터 보존하고, DNS 단일 장애점과 중단 영향을 확인합니다.

복구 검증은 A·SOA·외부 회선·필요한 내부 재귀·UDP/TCP·웹서비스를 구분합니다. 확정 원인과 추정 원인을 따로 기록합니다.

[로그 관리](05-logging.md) · [시간 점검](../server/linux/time-check.md) · [장애 양식](../../templates/incident-report.md)
