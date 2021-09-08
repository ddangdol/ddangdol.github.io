---
layout: posts
title: ubuntu 의 apt 사용시 lock 발생 원인과 해결책을 찾아보자
tags: [aws, eventbridge, event, rule]
sitemap :
  priority : 1.0
---

ubuntu 서버에서 apt install 사용 시 apt lock 이 발생하며 설치가 실패하는 경우가 간헐적으로 발생합니다. 저의 경우 ubuntu 이미지를 기반으로 진행되는 초반 provisiong 시점에 자주 발생했습니다.
```shell
[stdout]Waiting for cache lock: Could not get lock /var/lib/dpkg/lock-frontend. It is held by process 2737 (unattended-upgr)...
```
출력 메시지를 보면 unattended-upgrade 프로세스가 특정 패키지 업그레이드 작업을 하면서 lock 을 잡고 있는 것으로 추측됩니다. 그렇다면 이와 같은 현상이 왜 간헐적으로 발생하며 특히 provisionig 초반에 문제가 되는지 의문스럽습니다. 

앞으로 발생 원인 파악과 해결책을 알아보겠습니다.

---

## unattended-upgrade
ubuntu 는 정기적인 패키지 업그레이드 및 클린 작업을 위해 `unattended-upgrade` 패키지가 설치되어 있고, 이를 매일 동작시키기 위한 서비스와 스케쥴이 `systemd` 에 등록되어 있습니다.

unattended-upgrade 가 문제가 된다면 제거 또는 관련 스케쥴을 제거하면 되겠지만, 필수 보안 패키지의 주기적인 업그레이드를 위해 유지가 필요한 경우가 있습니다. 

---

## apt-daily.service & apt-daily-upgrade.service
위에서 설명한 `unattended-upgrdae` 를 꾸준히 동작시키기 위한 서비스들이 `systemd` 에 등록되어 있습니다.
```shell
$ systemctl list-unit-files | grep apt-daily
apt-daily-upgrade.service           static          enabled
apt-daily.service                   static          enabled
```
`apt-daily.service` 는 각종 패키지의 업데이트 및 다운로드를 담당합니다. 이후 업데이트된 패키지를 `apt-daily-upgrade.service` 가 업그레이드하고 필요없는 패키지들을 삭제하는 역할을 합니다.

---

## apt-daily.timer & apt-daily-upgrade.timer
위 두 서비스들은 `systemd` 의 타이머를 통해 스케쥴링 됩니다. 각 서비스별 타이머는 `apt-daily.timer` 와 `apt-daily-upgrade.timer` 로 서비스 이름과 동일합니다.
```shell
$ systemctl list-units | grep apt-daily
apt-daily-upgrade.timer            loaded active waiting   Daily apt upgrade and clean activities
apt-daily.timer                    loaded active waiting   Daily apt download activities
```
두 timer 의 기본 설정을 살펴보겠습니다.
```shell
# /etc/systemd/system/timers.target.wants/apt-daily.timer
[Unit]
Description=Daily apt download activities

[Timer]
OnCalendar=*-*-* 6,18:00
RandomizedDelaySec=12h
Persistent=true

[Install]
WantedBy=timers.target
```
`apt-daily.timer` 는 `OnCalendar` 와 `RandomizedDelaySec` 설정으로 6시 ~ 18시, 18시 ~ 6시 사이에 랜덤하게 동작합니다.

```shell
# /etc/systemd/system/timers.target.wants/apt-daily-upgrade.timer
[Unit]
Description=Daily apt upgrade and clean activities
After=apt-daily.timer

[Timer]
OnCalendar=*-*-* 6:00
RandomizedDelaySec=60m
Persistent=true

[Install]
WantedBy=timers.target
```
`apt-daily-upgrade.timer` 는 `OnCalendar` 와 `RandomizedDelaySec` 설정으로 6시 ~ 7시 사이에 랜덤하게 동작합니다.
다만 주의할 점은 `After=apt-daily.timer` 설정으로 사전조건으로 `apt-daily.timer` 동작 후 실행되도록 구성됩니다.

두 timer 모두 unit 에 등록되어 설정된 스케쥴 외 부팅 시에도 실행됩니다.
```shell
$ systemd-analyze blame | grep apt
10.332s apt-daily.service
 1.954s apt-daily-upgrade.service
```
systemd-analyze 를 통해 부팅시 실행된 프로세스의 실행시간과 관계들을 알 수 있습니다. 아래 명령을 통해 실행내역을 시각화할 수 있습니다.
```shell
$ systemd-analyze plot > bootup.svg
```

---

## apt 설치 시 lock 이 발생하는 이유
위에서 설명한 대로 apt-daily-upgrade.service 가 부팅 시 동작하게 됩니다.
```shell
# /var/log/syslog

...
systemd[1]: Starting Daily apt upgrade and clean activities...
...
```
해당 서비스는 unattended-upgrade 를 통해 패키지 업그레이드를 수행합니다.
```shell
# /var/log/apt/history.log

Start-Date: 2021-09-01  06:53:26
Commandline: /usr/bin/unattended-upgrade
Upgrade: ntfs-3g:amd64 (1:2017.3.23AR.3-3ubuntu1, 1:2017.3.23AR.3-3ubuntu1.1), libntfs-3g883:amd64 (1:2017.3.23AR.3-3ubuntu1, 1:2017.3.23AR.3-3ubuntu1.1)
End-Date: 2021-09-01  06:53:37

Start-Date: 2021-09-01  06:53:40
Commandline: /usr/bin/unattended-upgrade
Upgrade: squashfs-tools:amd64 (1:4.4-1, 1:4.4-1ubuntu0.1)
End-Date: 2021-09-01  06:53:40
...
```
패키지 업그레이드가 일시적인 네트워크 영향이나 오래된 AMI 를 사용하여 업그레이드 대상이 많을 경우 예상한 시간보다 작업 시간이 늘어나게 되고 그만큼 오랫동안 lock 을 잡게 됩니다. 
사용자의 수동 작업의 경우 크게 문제가 안됩니다. 물론 업그레이드 스케쥴과 겹쳐 진행이 안될 수 있으나 lock 을 해소하기 위한 작업 또는 lock 이 풀린 후 작업하게 되면 문제가 없습니다.

AMI 를 통한 provisionig 에서는 얘기가 달라집니다. 간헐적으로 배포시간이 늘어나거나 보통 배포시 스크립트이 타임아웃 설정을 하게 되는데 이로 인해 실패하게 되는 경우도 발생합니다. 의미없는 실패로 인해 운영을 위한 리소스가 소모되고 불필요한 알람등을 받게 됩니다.

---

## 부팅 시 apt-daily.timer 의 비활성화
`apt-daily-upgrade.timer` 는 `systemd` 의 `unit` 으로 등록되어 부팅시 자동실행됩니다. `unattended-upgrade` 의 정기적인 패키지 업그레이드 기능을 유지하면서 `apt lock` 을 피하기 위해 부팅시에 동작하지 않도록 `timer` 설정을 시도해볼 수 있습니다.

`systemd timer` 에는 `OnBootSec` 설정이 있습니다. 부팅 후 특정 시간동안 스케줄이 동작하지 않도록 설정할 수 있습니다. 아래 설정은 부팅 후 60분 동안의 스케쥴은 동작하지 않도록 조정됩니다.
```shell
[Timer]
OnBootSec=60min
```
`apt-daily.timer` 의 `OnBootSec` 설정만 추가해도 `api-daily-upgrade.timer` 는 부팅시 동작하지 않습니다. 선행조건인 `After=apt-daily.timer` 설정때문입니다.

apt-daily.timer 에 위 내용을 적용해봅니다.
```shell
$ systemctl edit apt-daily.timer
```
위 커맨드를 실행하면 에디터가 실행되고 기본 설정에 오버라이드할 내용들을 아래와 작성할 수 있습니다.
```shell
[Timer]
OnBootSec=60min
OnCalendar=*-*-* 6,18:00
RandomizedDelaySec=12h
Persistent=true
```
작성 후 저장하게 되면 `/etc/systemd/system/apt-daily.timer.d/override.conf` 경로에 저장됩니다.
```shell
$ systemctl daemon-reload
```
`daemon-reload` 로 변경된 설정을 적용합니다.

---

## 의도한대로 동작하는지 최종 확인
부팅 시 `apt-daily.timer` 가 동작하지 않도록 구성한 후 재부팅을 해봅시다. AWS EC2 인스턴스라면 stop 후 start 합니다.
```shell
$ systemd-analyze blame | grep apt
```
이제 부팅 시 실행내역에 apt-daily-upgrade.timer 가 조회되지 않습니다. AMI 기반의 provisionig 작업에서 초기충돌이 해결되고 부팅시간도 빨라지는 효과를 누릴 수 있습니다.

그렇다면 부팅 시 실행은 무시되지만 daily 스케쥴이 제대로 진행되는지 확인해보겠습니다.
```shell
$ systemctl list-timers | grep apt
Wed 2021-09-08 21:40:36 UTC 9h left       Wed 2021-09-08 10:50:23 UTC 54min ago    apt-daily.timer              apt-daily.service
Thu 2021-09-09 06:53:28 UTC 19h left      Wed 2021-09-08 06:51:54 UTC 4h 53min ago apt-daily-upgrade.timer      apt-daily-upgrade.service
```

timer 설정대로 다음 스케쥴이 잡힌 것을 알 수 있습니다. 실제 해당 시간 도달시 동작하는 것도 확인할 수 있습니다.

---

## 결론

구글링을 해보면 이와 같은 apt lock 충돌 상황을 생각보다 많은 사람들이 자주 겪는 것을 알 수 있었습니다. 다만 상황 발생 시 해당 기능을 disable 시키는 해결책들이 많았습니다. 물론 필요없는 경우도 있겠지만 최근 보안 이슈를 위해서라도 최소한의 주기적인 업그레이드를 위해 unattended-upgrade 를 위한 스케쥴을 살려두는 것이 좋다고 판단했습니다.

무작정 원인을 해결하기보다 동작원리를 이해하고 자신이 바라는 최선의 결과를 위해 조금은 깊게 들여다 보는 것이 올바른 길이라고 생각합니다.