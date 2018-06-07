bosh-lite 장애 Case 1
================================================================================

bosh cli 로 아예 director 에 connection refused 가 났기 때문에 director 에 문제가 생긴 것으로 추정함.
director 는 bosh-lite 의 경우 Vagrant VM 그 자체이므로 다음과 같이 로그인 할 수 있다.

.. code-block:: bash

    $ vagrant ssh

director 로그에는 access 자체가 안되는 것으로 보여서 시스템 로그를 확인해 보았다.

.. code-block:: text

  tail -f /var/log/syslog.log

  May 15 07:36:03 MY-SVR anacron[26411]: Job `cron.daily' terminated
  May 15 07:36:03 MY-SVR anacron[26411]: Normal exit (1 job run)
  May 15 08:17:01 MY-SVR CRON[26579]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
  May 15 09:17:01 MY-SVR CRON[26596]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
  May 15 10:17:01 MY-SVR CRON[26852]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
  May 15 10:23:23 MY-SVR systemd[1]: snapd.refresh.timer: Adding 26min 19.000282s random time.
  May 15 11:17:01 MY-SVR CRON[27041]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
  May 15 11:27:23 MY-SVR systemd[1]: snapd.refresh.timer: Adding 4h 39min 57.220974s random time.
  May 15 12:11:23 MY-SVR systemd[1]: Starting Daily apt activities...
  May 15 12:11:40 MY-SVR systemd[1]: Started Daily apt activities.
  May 15 12:11:40 MY-SVR systemd[1]: apt-daily.timer: Adding 11h 32min 45.317937s random time.
  May 15 12:11:40 MY-SVR systemd[1]: apt-daily.timer: Adding 9h 39min 19.081080s random time.
  May 15 12:17:01 MY-SVR CRON[27412]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)
  May 15 13:17:01 MY-SVR CRON[27469]: (root) CMD (   cd / && run-parts --report /etc/cron.hourly)

  May 15 13:19:36 MY-SVR kernel: [2146474.509021] device vboxnet0 left promiscuous mode (<-- 요기 주목!!!)
  May 15 13:19:36 MY-SVR NetworkManager[1233]: <info>  [1494821976.7642] device (vboxnet0): link disconnected
  May 15 13:19:36 MY-SVR kernel: [2146474.525511] vboxnetflt: 3649851 out of 3651331 packets were not sent (directed to host)

  May 15 13:24:02 MY-SVR systemd[1]: Started Session 781 of user lgcns.
  May 15 13:26:39 MY-SVR systemd[1]: Started Session 782 of user lgcns.
  May 15 13:26:39 MY-SVR systemd[1]: Started Session 783 of user lgcns.
  May 15 13:31:44 MY-SVR kernel: [2147202.365199] ip_tables: (C) 2000-2006 Netfilter Core Team

로그를 보았을 때 vboxnet0 라는 NIC 디바이스가 장애가 난 것으로 보인다. 

vboxnet0 는 Virtual Box 가 Host 와 Guest VM 사이의 통신을 위해 만든 가상 디바이스이다. 다음 명령으로 디바이스 리스트를 볼 수 있다.

.. code-block: bash

  $
  $
  $ VBoxManage list bridgedifs
  Name:            en1
  GUID:            xxx-xxx-xxx-xxx
  DHCP:            Disabled
  IPAddress:       00.00.00.00
  NetworkMask:     255.255.255.0
  IPV6Address:     ...
  IPV6NetworkMaskPrefixLength: 64
  HardwareAddress: 00:00:00:00:00:00
  MediumType:      Ethernet
  Status:          Up
  VBoxNetworkName: HostInterfaceNetworking-en1

  $
  $
  $ VBoxManage list hostonlyifs
  Name:            vboxnet0
  GUID:            xxx-xxx-xxx-xxx
  DHCP:            Disabled
  IPAddress:       192.168.50.1
  NetworkMask:     255.255.255.0
  IPV6Address:     ...
  IPV6NetworkMaskPrefixLength: 64
  HardwareAddress: 00:00:00:00:00:00
  MediumType:      Ethernet
  Status:          Up
  VBoxNetworkName: HostInterfaceNetworking-vboxnet0

이 경우 장치 오류이므로 어찌할 도리가 없어 서버를 재기동 하고 snapshot 으로 복구하였다.

관련 자료:

- http://www.linuxquestions.org/questions/linux-security-4/device-entered-promiscuous-mode-696873/
- https://forums.virtualbox.org/viewtopic.php?f=7&t=82378
