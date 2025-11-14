# Locket

https://github.com/cloudfoundry/locket

## Overview

Locket 은 grpc 서버이다. golang 으로 작성되었으며 [LocketClient][1] 를 통해 커뮤니케이션 할 수 있다. grpc 의 통신 프로토콜이 ProtocolBuffer 이므로 다른 언어로 통신하려면 [.proto files][2] 로 client 를 생성할 수 있다.


## Lfrit Runners

[lfrit][3] 은 단일 목적의 작업 단위로 더 큰 프로그램으로 구성하기 위한 작은 Interface 세트이다. 개발자는 자신의 프로그램을 하나의 작은 단위로 나누 후 각 작업 단위에서 Runner 인터페이스를 구현한다. 각 Runner Interface 를 호출함으로써 모니터하고 중지 신호를 보낼 수 있는 프로세스를 만들 수 있다. 

## Locket lock runner

[LockRunner][4] 는 lock 을 획득하기 위해 사용된다. 주위할 점은 이 Runner 는 lock 이 획득되기 전까지는 준비되지 않지만, lock 이 소실(lost) 된 순간 즉시 종료 (exit) 한다.

## Locket presence runner

[PresenceRunner][5] 는 현존하는 Service 를 등록 (Register) 하는데 사용된다. PresenceRunner 와 LockRunner 의 차이는 PresenceRunner 는 lock 이 소실 (lost) 될 경우 종료 (exit) 하는 대신 background 로 retry 를 한다는 접이다.

[1]: https://godoc.org/code.cloudfoundry.org/locket/models#LocketClient
[2]: https://github.com/cloudfoundry/locket/blob/master/models/locket.proto
[3]: https://github.com/tedsuo/ifrit
[4]: https://godoc.org/code.cloudfoundry.org/locket/lock#NewLockRunner
[5]: https://godoc.org/code.cloudfoundry.org/locket/lock#NewPresenceRunner