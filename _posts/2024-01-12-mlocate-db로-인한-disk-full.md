---
title: mlocate db로 인한 disk full
categories:
  - Linux
tags:
  - Linux
published: true
last_modified_at: 2024-01-19 (금) 15:25:32
created: 2024-01-18 (Thu목) 12:15:50
modified: 2024-01-19 (Fri금) 615:51:56
---

## 이슈

- 매일 새벽 시간대 특정 서버 루트(/) 디렉토리의 디스크 풀 알람이 발생했다가 해소되는 현상이 발생했다.
- 루트 디렉토리이기 때문에 지속 발생 시 서비스에 영향을 줄 수 있으므로 원인을 파악하기 시작했다.

## 원인 디렉토리 및 파일

- 디스크 풀의 원인이 되는 디렉토리 및 파일을 알아내기 위해 아래 스크립트를 크론탭에 등록하여 로그에 기록했다.
	- 매분 해당 스크립트를 통해 디스크 사용율이 높은 디렉토리를 기록했다.
	- (디스크가 계속 차있었다면, 해당 시점에 용량을 차지 하고 있는 디렉토리를 확인하면 됐을텐데, 이번 경우에는 디스크가 찼다가 다시 줄어드는 패턴이었기 때문에 문제 인지 시점에는 문제가 된 디렉토리를 알 수가 없었다.)
```bash
sudo du -h --max-depth=2 --exclude={proc,home1} / | grep -vE "^0\s" | sort -hr
```

- 확인 결과 아래와 같은 패턴이 발견되었다.
	- 03시 19분부터 `/var/lib` 디렉토리의 디스크 사용량이 급증하기 시작
	- 03시 29분까지 지속
	- 03시 33분에 정상화
- 결국 문제가 된 `/var/lib` 디렉토리를 살펴보았다.
- ![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/c3e64cf2-5243-4ffd-a1a2-af847df25b9a)
- ![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/31242dbe-09d4-45ae-892e-b008a84c4414)
- 살펴보니 `/var/lib/mlocate` 디렉토리의 modified time이 03시 32분(디스크 사용량 정상화시점)과 일치했고, 해당 디렉토리에는 7.5GB 크기의 `mlocate.db` 파일 하나만 존재했다.
- 그래서 이 파일의 용도가 무엇이며, 어떻게 만들어지는 것인지 추적하기 시작했다.

## mlocate.db
- `mlocate.db`는 리눅스 `locate` 유틸을 위한 색인용 데이터베이스이다.
	- ![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/e04706aa-a7a7-4db6-98a3-bda0df4bd2ec)
	- `locate`는 `find`와 유사한 용도의 유틸이며, `find`와의 차이점은 아래 [find와 locate의 차이점](#find와-locate의-차이점) 참고.
- 그렇다면 이 db 파일을 만드는 쪽은 어디인가?
	- ![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/71a8831b-02fe-49d6-b612-1a333e67400b)
	- 해당 `mlocate.db`는 `updatedb` 프로세스에 의해 생성되며, `updatedb`는 `cron daily`에 의해 하루에 한번 실행된다.
- 그래서 `cron.daily`를 확인해보니 `mlocate` 스크립트가 있었다.
	- 그리고 `mlocate` 스크립트 내부를 확인해보니 실제로는 `updatedb` 바이너리를 실행하는 것이었다.

```bash
[root@server ~]$ ls -al /etc/cron.daily/
total 24
drwxr-xr-x.  2 root root   57 Jul  1  2022 .
drwxr-xr-x. 94 root root 8192 Aug 23  2022 ..
-rwx------.  1 root root  219 Apr  1  2020 logrotate
-rwxr-xr-x.  1 root root  618 Oct 30  2018 man-db.cron
-rwx------.  1 root root  208 Apr 11  2018 mlocate
[root@server ~]$ sudo cat /etc/cron.daily/mlocate
#!/bin/sh
nodevs=$(awk '$1 == "nodev" && $2 != "rootfs" && $2 != "zfs" { print $2 }' < /proc/filesystems)

renice +19 -p $$ >/dev/null 2>&1
ionice -c2 -n7 -p $$ >/dev/null 2>&1
/usr/bin/updatedb -f "$nodevs"
```


- 그리고 실제 크론에 의해 수행되는 게 맞는지 확인하기 위해 크론 로그를 보니 약 새벽 3시경 실행이 됐던 것으로 확인했고,
- **`mlocate` 수행 시작 시점(03:19)과 수행 종료(03:32) 시점이 디스크 풀 발생 및 해소시간과 일치함을 발견했다.**
```
[root@server ~]$ sudo grep 'cron.daily' /var/log/cron
(생략)
Mar 29 03:01:01 server anacron[15318]: Will run job `cron.daily' in 18 min.
Mar 29 03:19:01 server anacron[15318]: Job `cron.daily' started
Mar 29 03:19:01 server run-parts(/etc/cron.daily)[29377]: starting logrotate
Mar 29 03:19:01 server run-parts(/etc/cron.daily)[29385]: finished logrotate
Mar 29 03:19:01 server run-parts(/etc/cron.daily)[29377]: starting man-db.cron
Mar 29 03:19:01 server run-parts(/etc/cron.daily)[29399]: finished man-db.cron
Mar 29 03:19:01 server run-parts(/etc/cron.daily)[29377]: starting mlocate
Mar 29 03:32:52 server run-parts(/etc/cron.daily)[26074]: finished mlocate
Mar 29 03:32:52 server anacron[15318]: Job `cron.daily' terminated
```
- 결국 `updatedb` 프로세스가 디스크풀을 유발 및 해소시키는 원인으로 가장 의심스러운 상황이었다.

## 그럼 왜 디스크풀이 발생할까?
- `updatedb` 프로세스는 색인 파일(`mlocate.db`)을 갱신하는 프로세스이다.
- 디스크 풀이 발생했다가 자연해소되는 패턴으로 보아, temp db 파일을 만들고, mv하는 방식으로 동작하지 않을까라고 예상하고 실제 소스코드를 확인해봤다.
- 실제 소스코드를 확인해보니 예상과 동일하게 temp파일을 만들고 rename하는 방식으로 동작하는 것을 확인했다.
	- ![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/618d997c-7e13-456b-98b7-132992398a1d)
		- [mlocate/src/updatedb.c at 0ce05077df942995aef99d1d009b7fa4372dc96c · msekletar/mlocate · GitHub](https://github.com/msekletar/mlocate/blob/0ce05077df942995aef99d1d009b7fa4372dc96c/src/updatedb.c#L912-L917)
	- ![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/0d186f6d-edab-44fd-a9e6-36c2ad49cede)
		- [mlocate/src/updatedb.c at 0ce05077df942995aef99d1d009b7fa4372dc96c · msekletar/mlocate · GitHub](https://github.com/msekletar/mlocate/blob/0ce05077df942995aef99d1d009b7fa4372dc96c/src/updatedb.c#L1018)
- 결국 `updatedb` 프로세스가 수행되는 도중에는 실제 색인 파일 + 임시 색인 파일이 동시에 존재하게 되어, 색인 파일크기의 최대 2배 가까이 디스크를 사용하면서 디스크 풀이 발생한 것이고
- `updatedb` 갱신 완료 시점에는 하나의 파일만 남게 되어 디스크 사용량이 다시 줄어들어드는 것이다.

## 해결방안

근본적인 원인은 색인 파일인 `mlocate.db` 파일 사이즈가 큰 것이 원인이고, 결국 이 색 인 파일의 크기는 전체 디스크 사용율에 비례한다.
문제가 된 서버의 경우도, 실제로 1TB 이상을 사용하고 있던 서버였기 때문에 색인 파일 크기도 그에 따라 커졌던 것이다.
그렇기 때문에 전체 파일시스템 중 사용하지 않는 파일을 지울 수 있다면, `mlocate.db` 파일 사이즈도 줄어들어 자연스럽게 해결될 수 있다.

하지만 우리의 경우에는 실제로 해당 파일들이 모두 사용하고 있는 파일들이었기 때문에 디스크 사용율은 유지하면서 간단하게 해결할 수 있는 방안을 검토했다.

1. `/etc/updatedb.conf`의 `PRUNEPATHS`에 특정 디렉토리 추가
	- `/etc/updatedb.conf`는 이름에서도 알 수 있듯이 `updatedb` 프로세스의 설정 정보를 담고 있는 파일이다.
	- 그리고 설정 중에는 `PRUNEPATHS` 환경 변수를 통해서 지정할 수 있는 설정이 있는데, 이 설정은 색인에서 제외할 수 있는 경로들을 지정할 수 있는 설정이다.
	- 그렇다면, 디스크 사용율이 높지만 `locate`에서 참조될 일이 없는 경로를 색인에서 제외하면, `mlocate.db` 파일이 커지는 문제를 해결할 수 있다.
	- ![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/78479de9-2ed0-4e37-8df0-47f7ccff2c1d)
2. `updatedb`의 옵션(`output`)을 통해 db 경로 변경
	- ![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/836ccd86-0fc6-4dbd-b7a7-2070517283f4)
	- `updatedb` 스크립트의 옵션에 결과 색인 파일명을 지정할 수 있는 옵션(`-o`)이 있다.
	- 지정하지 않는 경우 디폴트(`/var/lib/mlocate/mlocate.db`)가 된다.
	- 해당 옵션을 통해 마운트된 크기가 큰 디스크 영역에 색인 파일을 저장한다면, 당장 문제가 된 디스크 영역의 디스크 풀은 해결할 수 있다.
	- 다만, 색인 파일의 크기가 큰 것은 여전히 남아있는 문제이다.
3. `cron.daily`에서 `mlocate`를 제외
	- 사실 서비스와 관련 없는 리눅스의 유틸이기 때문에 주기적으로 색인파일을 업데이트해줄 필요가 없어보였다.

3가지 안 중 우리는 1안을 적용했다.

3안은 `locate`에 의존적인 프로세스의 존재 여부에 대한 조사가 필요했기 때문에 제외를 했고, 1안과 2안 중 좀 더 근본적으로 색인 파일의 크기를 줄일 수 있는 1안을 채택 및 적용했다.

1안을 적용 후 수동으로 `updatedb` 명령어를 수행한 결과,

![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/5efbc23c-bdcf-465e-b087-acd83b518241)

7.5G에 달했던 색인 파일이 1.9M까지 줄어들었다.

## 그외

### find와 locate의 차이점
일반적으로 리눅스 환경에서 조건에 맞는 파일을 찾을때 `find` 명령어를 많이 사용했는데, 이번에 `locate`도 동일한 기능을 하는 것을 알게됐다.
그렇다면 두 가지 유틸의 차이는 무엇일지 궁금하여 찾아보았다.

`find`는 명령을 수행한 시점에 파일시스템을 직접 탐색하는 반면, `locate`는 사전에 색인을 만들어두고(`cron.daily`에 의해 하루에 한번) 색인에 기록된 내용을 바탕으로 결과를 알려준다.
그렇기 때문에, 조회 속도 관점에서는 `locate`가 빠를 수 있지만, 정보의 실시간성을 반영하지 못하는 문제가 있다.
결국 일반적인 db에 직접 검색 vs 색인을 통한 검색의 트레이드 오프를 그대로 가져간다고 볼 수 있다.

---

## References
- [GitHub - msekletar/mlocate: mlocate hacking](https://github.com/msekletar/mlocate)
- [updatedb(8): update database for mlocate - Linux man page](https://linux.die.net/man/8/updatedb)
- [locate(1): find files by name - Linux man page](https://linux.die.net/man/1/locate)
- [updatedb.conf(5): config file for updatedb - Linux man page](https://linux.die.net/man/5/updatedb.conf)
- [[Linux] Locate 와 FIND 명령어 차이점](https://anggeum.tistory.com/entry/Linux-Locate-%EC%99%80-FIND-%EB%AA%85%EB%A0%B9%EC%96%B4-%EC%B0%A8%EC%9D%B4%EC%A0%90)
