# BOSH task

BOSH 를 통한 모든 작업은 task 라고 하는 단위로 관리 된다. 그리고 task 를 조회하는 작업도 하나의 task 이므로 아래와 같이 watch 명령으로 반복 조회를 할 경우 어느 순간 task number 가 순식간이 올라가 있는 것을 발견하게 될 것이다. 조심하자.

## Tasks
```
    $ watch bosh -e lite tasks
    Using environment '192.168.50.4' as client 'admin'

    #    State   Started At                    Last Activity At              User       Deployment  Description                    Result
    256  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 07:00:00 UTC 2017  scheduler  -           scheduled SnapshotDeployments  -
    255  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:53:46 UTC 2017  admin      cf-mysql    delete deployment cf-mysql     -
    254  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:53:26 UTC 2017  admin      cf-mysql    retrieve vm-stats              -
    253  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:52:38 UTC 2017  admin      cf          retrieve vm-stats              -
    252  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:49:39 UTC 2017  admin      cf-mysql    delete deployment cf-mysql     -
    251  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:49:14 UTC 2017  admin      cf          retrieve vm-stats              -
    250  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:48:34 UTC 2017  admin      cf          retrieve vm-stats              -
    249  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:48:16 UTC 2017  admin      cf-mysql    delete deployment cf-mysql     -
    245  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:25:53 UTC 2017  admin      cf-mysql    retrieve vm-stats              -
    244  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:24:50 UTC 2017  admin      cf-mysql    retrieve vm-stats              -
    243  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:24:20 UTC 2017  admin      cf          retrieve vm-stats              -
    242  queued  Thu Jan  1 00:00:00 UTC 1970  Mon May 22 06:22:43 UTC 2017  admin      cf          retrieve vm-stats              -

    12 tasks

    Succeeded
```

## cancel-task

```
    $ bosh -e lite cancel-task 256
```
이렇게 하여 진행중인 task 를 취소할 수 있다. 이것은 때때로 무한히 대기를 탈 수도 있다.