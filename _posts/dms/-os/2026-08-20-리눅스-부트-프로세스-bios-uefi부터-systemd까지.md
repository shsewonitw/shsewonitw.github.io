---
layout: post
title: "[Daily morning study] 리눅스 부트 프로세스: BIOS/UEFI부터 systemd까지"
description: >
  #daily morning study
category: 
    - dms
    - dms-os
hide_last_modified: true
---

![Image](https://github.com/user-attachments/assets/1b38c764-1122-4c72-8acb-ac3a67750ee9)

---

컴퓨터 전원 버튼을 누르는 순간부터 로그인 프롬프트가 뜰 때까지, 리눅스 시스템은 여러 단계를 거쳐 부팅된다. 각 단계가 어떤 역할을 하는지 순서대로 정리해봤다.

## 부트 프로세스 전체 흐름

```
전원 ON
  └→ BIOS/UEFI (POST)
       └→ 부트로더 (GRUB2)
            └→ 커널 로딩
                 └→ initramfs
                      └→ 루트 파일시스템 마운트
                           └→ init 프로세스 (systemd)
                                └→ 로그인 프롬프트
```

---

## 1단계: BIOS / UEFI

전원이 켜지면 CPU는 ROM에 저장된 펌웨어 코드를 실행한다. 여기서 두 가지 방식이 있다.

### BIOS (Basic Input/Output System)

- 오래된 방식 (16비트 리얼 모드)
- **POST (Power-On Self-Test)** 수행: CPU, RAM, 디스크, 키보드 등 하드웨어 이상 유무 체크
- MBR(Master Boot Record)에서 부트로더를 찾아 실행

**MBR 구조 (512바이트)**

| 영역 | 크기 | 내용 |
|------|------|------|
| 부트로더 코드 | 446 bytes | 1단계 부트로더 (GRUB stage 1) |
| 파티션 테이블 | 64 bytes | 최대 4개 주 파티션 정보 |
| 매직 넘버 | 2 bytes | `0x55AA` (유효한 MBR 표시) |

MBR은 크기가 너무 작아서 446바이트에 전체 부트로더를 담지 못한다. 그래서 GRUB는 `stage 1 → stage 1.5 → stage 2` 방식으로 나뉘어 로딩된다.

### UEFI (Unified Extensible Firmware Interface)

- 현대적인 방식 (32/64비트, C 언어 기반)
- MBR 대신 **GPT(GUID Partition Table)** 사용 → 2TB 이상 디스크 지원
- **ESP(EFI System Partition)** 라는 FAT32 파티션에서 `.efi` 형태의 부트로더를 직접 실행
- Secure Boot 지원: 서명되지 않은 코드 실행 차단

BIOS가 MBR을 통해 간접적으로 부트로더를 찾는 반면, UEFI는 ESP 파티션에서 직접 `.efi` 파일을 실행한다.

---

## 2단계: 부트로더 (GRUB2)

현재 리눅스 대부분은 **GRUB2 (GRand Unified Bootloader version 2)**를 사용한다.

GRUB2의 역할:
1. 커널 이미지(`vmlinuz`) 메모리에 로드
2. initramfs(초기 램 파일시스템) 로드
3. 커널에 부팅 파라미터 전달
4. 커널 실행

GRUB 설정 파일은 `/boot/grub/grub.cfg`이고, 이걸 직접 수정하는 대신 `/etc/default/grub`과 `/etc/grub.d/` 스크립트를 수정한 뒤 `update-grub` 명령으로 재생성한다.

```bash
# grub.cfg 재생성
sudo update-grub           # Debian/Ubuntu
sudo grub2-mkconfig -o /boot/grub2/grub.cfg   # RHEL/Fedora
```

GRUB 부팅 화면에서 `e`를 누르면 커널 파라미터를 임시로 수정할 수 있다. 복구 모드 진입(`rd.break`, `init=/bin/bash`)에 자주 쓰인다.

---

## 3단계: 커널 로딩

GRUB가 커널 이미지(`/boot/vmlinuz-<version>`)를 메모리에 올리면 커널이 자체적으로 압축을 풀고 초기화를 시작한다.

커널 초기화 순서:
1. CPU, 메모리 구조 설정 (페이지 테이블, GDT, IDT 초기화)
2. 드라이버 및 서브시스템 초기화
3. `/sbin/init` (또는 `systemd`) 실행

커널은 `/proc`, `/sys`, `/dev` 같은 가상 파일시스템도 마운트한다. 이것들은 실제 디스크에 없고, 커널이 동적으로 생성하는 인터페이스다.

```
/proc  — 프로세스 및 커널 정보
/sys   — 장치, 드라이버, 커널 파라미터
/dev   — 장치 파일 (udev가 관리)
```

---

## 4단계: initramfs (초기 램 파일시스템)

커널이 루트 파일시스템(`/`)을 마운트하려면 디스크 드라이버가 필요한데, 그 드라이버가 아직 로드되지 않은 상태일 수 있다. 이 닭-달걀 문제를 해결하기 위해 **initramfs**가 존재한다.

initramfs(`/boot/initrd.img-<version>`)는:
- 압축된 cpio 아카이브로, 메모리에 임시 루트 파일시스템을 만든다.
- 필요한 드라이버 모듈과 유틸리티(busybox 등)를 포함
- 실제 루트 파일시스템을 마운트할 수 있게 되면 제어권을 넘긴다.

```bash
# initramfs 내용 확인
lsinitrd /boot/initrd.img-$(uname -r)
```

LVM, LUKS 암호화 디스크, RAID 같은 복잡한 스토리지 환경에서 initramfs의 역할이 특히 중요하다.

---

## 5단계: systemd (init 프로세스)

커널이 루트 파일시스템 마운트를 완료하면 **PID 1** 프로세스를 실행한다. 현대 리눅스 배포판은 대부분 `systemd`를 사용한다.

### systemd의 역할

- 병렬로 서비스(unit)를 시작해서 부팅 시간 단축
- 의존성 기반으로 서비스 순서 결정
- 서비스 실패 시 자동 재시작
- 저널(journald)로 로그 통합 관리

### Unit 타입

| 타입 | 확장자 | 설명 |
|------|--------|------|
| Service | `.service` | 데몬 프로세스 |
| Socket | `.socket` | 소켓 기반 활성화 |
| Target | `.target` | 논리적 그룹 (구 런레벨) |
| Mount | `.mount` | 파일시스템 마운트 포인트 |
| Timer | `.timer` | cron 대체 스케줄러 |

### Target (런레벨 대응)

SysV init의 런레벨(runlevel) 개념을 target이 대체한다.

| SysV 런레벨 | systemd target | 의미 |
|------------|----------------|------|
| 1 | `rescue.target` | 단일 사용자 모드 |
| 3 | `multi-user.target` | 텍스트 모드 멀티유저 |
| 5 | `graphical.target` | GUI 모드 |
| 6 | `reboot.target` | 재부팅 |

```bash
# 현재 기본 target 확인
systemctl get-default

# target 변경
sudo systemctl set-default multi-user.target

# 서비스 상태 확인 및 관리
systemctl status nginx
systemctl start/stop/restart/enable/disable nginx

# 부팅 로그 확인
journalctl -b           # 이번 부팅 전체 로그
journalctl -b -p err    # 에러 이상 로그만
```

---

## 부팅 시간 분석

`systemd-analyze`로 부팅 성능을 분석할 수 있다.

```bash
# 전체 부팅 시간
systemd-analyze

# 서비스별 시작 시간 확인
systemd-analyze blame

# 부팅 의존성 그래프 (SVG로 저장)
systemd-analyze plot > boot.svg
```

출력 예시:
```
Startup finished in 1.234s (kernel) + 2.345s (initrd) + 5.678s (userspace) = 9.257s
graphical.target reached after 5.600s in userspace
```

---

## 정리

| 단계 | 주체 | 핵심 역할 |
|------|------|----------|
| POST | BIOS/UEFI | 하드웨어 이상 체크, 부트 장치 탐색 |
| 부트로더 | GRUB2 | 커널과 initramfs를 메모리에 로드 |
| 커널 초기화 | Linux Kernel | 하드웨어 추상화, 가상 FS 마운트 |
| 초기 램디스크 | initramfs | 드라이버 로드, 실제 루트 FS 마운트 |
| init 시스템 | systemd | 서비스 시작, 사용자 환경 구성 |

전원을 켜는 순간부터 로그인 프롬프트까지 이 모든 과정이 수초 안에 일어난다. 부팅 문제를 디버깅하거나 시스템 복구 모드를 사용할 때 각 단계를 이해하고 있으면 어느 지점에서 실패했는지 빠르게 파악할 수 있다.
