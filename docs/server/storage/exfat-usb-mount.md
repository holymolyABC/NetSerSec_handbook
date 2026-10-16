# Linux 외장 디스크 exFAT 마운트 오류 진단

Linux 서버에서 USB 외장 디스크가 블록 장치로는 인식되지만 `mount` 시 `unknown filesystem type 'exfat'` 오류가 발생하는 경우의 점검 절차입니다.

## 증상

확인된 화면에서는 다음 상태였습니다.

- 시스템 디스크: `/dev/sda` 약 1.8T
- 약 931.5G 디스크가 `/dev/sdb`, `/dev/sdc`, `/dev/sdd`, `/dev/sde`로 여러 개 인식됨
- `/dev/sdc2`는 약 931.5G 파티션
- 마운트 시도:

```bash
mkdir -p /mnt/BackupDisk
mount /dev/sdc2 /mnt/BackupDisk
```

- 실제 오류:

```text
mount: unknown filesystem type 'exfat'
```

이 오류는 디스크 자체가 인식되지 않은 것이 아니라, 커널/사용자 공간에 exFAT 파일시스템 지원이 없거나 필요한 패키지가 설치되지 않은 상태일 때 주로 발생합니다.

## 1. 먼저 외장 디스크를 정확히 식별

사진에서는 931.5G 디스크가 여러 개 존재하므로, **`/dev/sdc`가 외장 디스크라고 단정하면 안 됩니다.** 포맷 전에 반드시 모델·시리얼·연결 버스를 확인합니다.

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN,FSTYPE,MOUNTPOINT
```

USB 외장 디스크라면 `TRAN`에 보통 `usb`가 표시됩니다.

개별 장치 확인:

```bash
udevadm info --query=property --name=/dev/sdc \
  | grep -E 'ID_BUS|ID_MODEL|ID_SERIAL'
```

예상 예:

```text
ID_BUS=usb
ID_MODEL=External_USB
ID_SERIAL=...
```

> `/dev/sdX` 이름은 재부팅이나 연결 순서에 따라 달라질 수 있습니다. 운영 절차에서는 모델·시리얼·UUID까지 함께 확인합니다.

## 2. 파일시스템 확인

```bash
blkid /dev/sdc2
```

또는:

```bash
lsblk -f
```

exFAT라면 예:

```text
/dev/sdc2: UUID="..." TYPE="exfat"
```

화면의 `unknown filesystem type 'exfat'` 메시지와 함께 보면, 파일시스템은 exFAT인데 현재 OS가 이를 마운트하지 못하는 상황으로 판단할 수 있습니다.

## 3. OS와 exFAT 지원 여부 확인

OS 버전:

```bash
cat /etc/redhat-release 2>/dev/null || cat /etc/os-release
uname -r
```

현재 커널의 exFAT 지원 여부:

```bash
grep exfat /proc/filesystems
```

모듈 존재 여부:

```bash
modinfo exfat 2>/dev/null
```

패키지 검색:

```bash
rpm -qa | grep -Ei 'exfat|fuse'
```

## 4. exFAT 지원 패키지 설치

배포판과 버전에 따라 패키지명이 다릅니다.

RHEL/CentOS 계열에서 저장소에 패키지가 있는 경우 예:

```bash
yum install exfat-fuse exfat-utils
```

신형 RHEL/Rocky 계열에서는 배포판 저장소 구성에 따라 다음 패키지가 사용될 수 있습니다.

```bash
dnf install exfatprogs
```

설치 가능 여부는 먼저 조회하는 것이 안전합니다.

```bash
yum list available '*exfat*'
# 또는
dnf list available '*exfat*'
```

> 레거시 CentOS/RHEL은 기본 저장소에 exFAT 패키지가 없을 수 있습니다. 운영 서버에서는 임의의 외부 저장소를 추가하기 전에 보안정책과 패키지 출처를 확인합니다.

## 5. 마운트 재시도

```bash
mkdir -p /mnt/BackupDisk
mount /dev/sdc2 /mnt/BackupDisk
```

확인:

```bash
df -h /mnt/BackupDisk
ls -al /mnt/BackupDisk
mount | grep '/mnt/BackupDisk'
```

쓰기 테스트가 허용되는 빈 백업 디스크라면:

```bash
touch /mnt/BackupDisk/.write-test
ls -l /mnt/BackupDisk/.write-test
rm -f /mnt/BackupDisk/.write-test
```

## 6. Linux 전용 백업 디스크라면 ext4 선택 가능

Windows/macOS와 공유할 필요가 없고 Linux 서버 백업용으로만 쓸 디스크라면 ext4가 운영상 단순합니다.

**주의: 다음 명령은 해당 파티션의 데이터를 삭제합니다. 실제 외장 디스크가 맞는지 모델·시리얼·UUID를 먼저 검증해야 합니다.**

```bash
mkfs.ext4 /dev/sdc2
```

이후:

```bash
mount /dev/sdc2 /mnt/BackupDisk
df -h /mnt/BackupDisk
```

데이터가 있는 외장 디스크에는 실행하지 않습니다.

## 7. 안전한 확인 순서

1. `lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN,FSTYPE,MOUNTPOINT`로 장치 식별
2. `blkid` 또는 `lsblk -f`로 파일시스템 확인
3. OS·커널·exFAT 지원 여부 확인
4. 필요한 경우 공식/승인 저장소의 exFAT 패키지 설치
5. 읽기/쓰기 마운트 검증
6. Linux 전용 디스크이고 기존 데이터를 삭제해도 되는 경우에만 ext4 포맷 검토

## 결과 해석

| 결과 | 의미 | 다음 조치 |
|---|---|---|
| `lsblk`에 디스크 없음 | USB/전원/케이블/컨트롤러 수준 문제 가능 | `dmesg`, USB 포트, 전원 확인 |
| 디스크는 보이고 `TYPE=exfat` | 장치 인식 정상 | exFAT 지원 여부 확인 |
| `unknown filesystem type 'exfat'` | OS에서 exFAT 드라이버/지원 미확인 | 커널·패키지 점검 |
| `wrong fs type, bad superblock` | 파일시스템 손상 또는 타입 오인 가능 | `blkid`, 파일시스템 검사 검토 |
| 마운트 성공, 쓰기 실패 | 읽기전용 마운트·권한·파일시스템 오류 가능 | `mount`, `dmesg`, 권한 확인 |

## 이번 작업의 검증 범위

**확인됨**

- 약 931.5G 디스크 여러 개가 OS에 인식됨
- `/dev/sdc2` 마운트 시 `unknown filesystem type 'exfat'` 오류 발생
- 따라서 블록 장치 인식 단계까지는 통과했음

**확인 필요**

- `/dev/sdc`가 실제 외장 디스크인지 여부
- 정확한 Linux 배포판/버전
- `/dev/sdc2`의 `blkid` 결과
- exFAT 패키지 설치 가능 여부
- 최종 마운트 성공 여부

미확인 상태에서 `mkfs`를 실행하지 않습니다.
