# 네트워크 운영

실제 새 작업을 다음 기준으로 분류합니다. 현재 특정 모델의 작업 완료를 기록한 것은 아닙니다.

| 분류 | 기록 항목 |
|---|---|
| L1·물리 | link·광레벨·CRC·케이블·SFP·flap |
| L2 | VLAN·trunk·STP·MAC·LACP·포트 보안 |
| L3 | IP·mask·gateway·ARP·route·이중화 |
| 무선 | AP/WLC·인증·RADIUS·전파·로밍 |
| 관리 접속 | console·SSH·관리IP·시간·syslog |

## IP에서 물리 포트까지 추적

서버 존재 여부를 ping만으로 판단하지 않습니다. 방화벽 NAT와 세션을 확인해 실제 목적지IP를 찾습니다. 같은 세그먼트 게이트웨이의 ARP로 MAC을 확인하고 스위치 MAC테이블에서 포트를 추적하여 배선·자산대장과 대조합니다. ARP가 없다는 이유만으로 서버가 없다고 확정하지 않습니다.

제품·펌웨어별로 조회와 변경 명령을 구분합니다. 제조사 미확인 장비에 Cisco 명령을 임의로 적용하지 않습니다.

[절차서 양식](../../templates/runbook.md) · [장애 보고 양식](../../templates/incident-report.md)
