# IaaS 네트워크 장비 교체 작업 후 mariadb galera cluster 장애

## 장애 상황

mysqld 프로세스를 띄우는 데 사용된 명령 라인

```
/var/vcap/packages/mariadb/bin/mysqld 
	--basedir=/var/vcap/packages/mariadb 
	--datadir=/var/vcap/store/mysql 
	--plugin-dir=/var/vcap/packages/mariadb/lib/plugin 
	--wsrep_on=ON 
	--wsrep_provider=/var/vcap/packages/mariadb/lib/plugin/libgalera_smm.so 
	--log-error=/var/vcap/sys/log/mysql/mysql.err.log 
	--pid-file=/var/vcap/sys/run/mysql/mysql.pid 
	--socket=/var/vcap/sys/run/mysql/mysqld.sock 
	--port=3306 
	--wsrep_start_position=746e46f3-8f76-11e7-9c1c-a220eabab4ba:20626756
```

명령 라인에 명시된 pid 파일(/var/vcap/sys/run/mysql/mysql.pid)이 존재하지 않음
명령 라인에 명시된 socket 파일(/var/vcap/sys/run/mysql/mysqld.sock) 이 존재함: 프로세스간 통신으로 DB 에 접근은 가능할 것으로 예상됨.

mysql VM 에서 서비스 중인 port 검사: 클러스터 리플리케이션에 사용되는 4567 포트는 정상이나 서비스 포트인 3306 은 사라짐

```
# netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.1:2822          0.0.0.0:*               LISTEN      1437/monit
tcp        0      0 127.0.0.1:2825          0.0.0.0:*               LISTEN      985/bosh-agent
tcp        0      0 0.0.0.0:9100            0.0.0.0:*               LISTEN      2038/node_exporter
tcp        0      0 0.0.0.0:51692           0.0.0.0:*               LISTEN      578/rpc.statd
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      566/rpcbind
tcp        0      0 0.0.0.0:9200            0.0.0.0:*               LISTEN      1935/galera-healthc
tcp        0      0 127.0.0.1:33331         0.0.0.0:*               LISTEN      985/bosh-agent
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      866/sshd
tcp        0      0 0.0.0.0:4567            0.0.0.0:*               LISTEN      1890/mysqld
udp        0      0 127.0.0.1:514           0.0.0.0:*                           1417/rsyslogd
udp        0      0 0.0.0.0:694             0.0.0.0:*                           566/rpcbind
udp        0      0 0.0.0.0:33474           0.0.0.0:*                           578/rpc.statd
udp        0      0 127.0.0.1:754           0.0.0.0:*                           578/rpc.statd
udp        0      0 0.0.0.0:22522           0.0.0.0:*                           668/dhclient
udp        0      0 0.0.0.0:68              0.0.0.0:*                           668/dhclient
udp        0      0 0.0.0.0:111             0.0.0.0:*                           566/rpcbind
```

명령 라인에 명시된 log 파일에서 IaaS 작업 시간대의 로그를 발췌하면 다음과 같다.
로그를 통해 알 수 있는 사실은 클러스터 복구 작업 도중 프로세스가 멈춰있다는 것이다. (이 상태로 Timeout 도 없이 2일간 지속됨)

```
mysql/4c1906d1-....:/var/vcap/sys/log/mysql# cat /var/vcap/sys/log/mysql/mysql.err.log | sed '1,554d'
171021 21:56:54 mysqld_safe Starting mysqld daemon with databases from /var/vcap/store/mysql
171021 21:56:54 mysqld_safe WSREP: Running position recovery with --log_error='/var/vcap/store/mysql/wsrep_recovery.dHPgeE' --pid-file='/var/vcap/store/mysql/9693da77-4ad5-4bd2-ae62-8eb421f87d1a-recover.pid'
2017-10-21 21:56:55 140111964899200 [Note] /var/vcap/packages/mariadb/bin/mysqld (mysqld 10.1.24-MariaDB) starting as process 1822 ...
2017-10-21 21:56:55 140111964899200 [Warning] An old style --language or -lc-message-dir value with language specific part detected: /var/vcap/packages/mariadb/share/english/
2017-10-21 21:56:55 140111964899200 [Warning] Use --lc-messages-dir without language specific part instead.
171021 21:57:32 mysqld_safe WSREP: Recovered position 746e46f3-8f76-11e7-9c1c-a220eabab4ba:20626756
2017-10-21 21:57:32 140046406944640 [Note] /var/vcap/packages/mariadb/bin/mysqld (mysqld 10.1.24-MariaDB) starting as process 1890 ...
2017-10-21 21:57:32 140046406944640 [Warning] An old style --language or -lc-message-dir value with language specific part detected: /var/vcap/packages/mariadb/share/english/
2017-10-21 21:57:32 140046406944640 [Warning] Use --lc-messages-dir without language specific part instead.
2017-10-21 21:57:32 140046406944640 [Note] WSREP: Read nil XID from storage engines, skipping position init
2017-10-21 21:57:32 140046406944640 [Note] WSREP: wsrep_load(): loading provider library '/var/vcap/packages/mariadb/lib/plugin/libgalera_smm.so'
2017-10-21 21:57:33 140046406944640 [Note] WSREP: wsrep_load(): Galera 3.20(rXXXX) by Codership Oy <info@codership.com> loaded successfully.
2017-10-21 21:57:33 140046406944640 [Note] WSREP: CRC-32C: using hardware acceleration.
2017-10-21 21:57:33 140046406944640 [Note] WSREP: Found saved state: 746e46f3-8f76-11e7-9c1c-a220eabab4ba:-1, safe_to_bootsrap: 0
2017-10-21 21:57:33 140046406944640 [Note] WSREP: Passing config to GCS: base_dir = /var/vcap/store/mysql/; base_host = 10.62.154.5; base_port = 4567; cert.log_conflicts = no; debug = no; evs.auto_evict = 0; evs.delay_margin = PT1S; evs.delayed_keep_period = PT30S; evs.inactive_check_period = PT0.5S; evs.inactive_timeout = PT15S; evs.join_retrans_period = PT1S; evs.max_install_timeouts = 3; evs.send_window = 4; evs.stats_report_period = PT1M; evs.suspect_timeout = PT5S; evs.user_send_window = 2; evs.view_forget_timeout = PT24H; gcache.dir = /var/vcap/store/mysql/; gcache.keep_pages_size = 0; gcache.mem_size = 0; gcache.name = /var/vcap/store/mysql//galera.cache; gcache.page_size = 128M; gcache.recover = no; gcache.size = 512M; gcomm.thread_prio = ; gcs.fc_debug = 0; gcs.fc_factor = 1.0; gcs.fc_limit = 16; gcs.fc_master_slave = no; gcs.max_packet_size = 64500; gcs.max_throttle = 0.25; gcs.recv_q_hard_limit = 9223372036854775807; gcs.recv_q_soft_limit = 0.25; gcs.sync_donor = no; gmcast.segment = 0; gmcast.version = 0; pc.announce_timeout = PT3S; pc.ch
2017-10-21 21:57:33 140046406944640 [Note] WSREP: GCache history reset: old(746e46f3-8f76-11e7-9c1c-a220eabab4ba:0) -> new(746e46f3-8f76-11e7-9c1c-a220eabab4ba:20626756)
2017-10-21 21:57:33 140046406944640 [Note] WSREP: Assign initial position for certification: 20626756, protocol version: -1
2017-10-21 21:57:33 140046406944640 [Note] WSREP: wsrep_sst_grab()
2017-10-21 21:57:33 140046406944640 [Note] WSREP: Start replication
2017-10-21 21:57:33 140046406944640 [Note] WSREP: Setting initial position to 746e46f3-8f76-11e7-9c1c-a220eabab4ba:20626756
2017-10-21 21:57:33 140046406944640 [Note] WSREP: protonet asio version 0
2017-10-21 21:57:33 140046406944640 [Note] WSREP: Using CRC-32C for message checksums.
2017-10-21 21:57:33 140046406944640 [Note] WSREP: backend: asio
2017-10-21 21:57:33 140046406944640 [Note] WSREP: gcomm thread scheduling priority set to other:0
2017-10-21 21:57:33 140046406944640 [Note] WSREP: restore pc from disk successfully
2017-10-21 21:57:33 140046406944640 [Note] WSREP: GMCast version 0
2017-10-21 21:57:33 140046406944640 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') listening at tcp://0.0.0.0:4567
2017-10-21 21:57:33 140046406944640 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') multicast: , ttl: 1
2017-10-21 21:57:33 140046406944640 [Note] WSREP: EVS version 0
2017-10-21 21:57:33 140046406944640 [Note] WSREP: gcomm: connecting to group 'cf-mariadb-galera-cluster', peer '10.62.154.4:,10.62.154.5:,10.62.154.13:'
2017-10-21 21:57:33 140046406944640 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') connection established to 069ddd4c tcp://10.62.154.5:4567
2017-10-21 21:57:33 140046406944640 [Warning] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') address 'tcp://10.62.154.5:4567' points to own listening address, blacklisting
2017-10-21 21:57:36 140046406944640 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') connection to peer 069ddd4c with addr tcp://10.62.154.5:4567 timed out, no messages seen in PT3S
2017-10-21 21:57:36 140046406944640 [Warning] WSREP: no nodes coming from prim view, prim not possible
2017-10-21 21:57:36 140046406944640 [Note] WSREP: view(view_id(NON_PRIM,069ddd4c,109) memb {
	069ddd4c,0
} joined {
} left {
} partitioned {
})
2017-10-21 21:57:36 140046406944640 [Note] WSREP: gcomm: connected
2017-10-21 21:57:36 140046406944640 [Note] WSREP: Changing maximum packet size to 64500, resulting msg size: 32636
2017-10-21 21:57:36 140046406944640 [Note] WSREP: Shifting CLOSED -> OPEN (TO: 0)
2017-10-21 21:57:36 140046406944640 [Note] WSREP: Opened channel 'cf-mariadb-galera-cluster'
2017-10-21 21:57:36 140046406944640 [Note] WSREP: Waiting for SST to complete.
2017-10-21 21:57:36 140045696608000 [Note] WSREP: New COMPONENT: primary = no, bootstrap = no, my_idx = 0, memb_num = 1
2017-10-21 21:57:36 140045696608000 [Note] WSREP: Flow-control interval: [16, 16]
2017-10-21 21:57:36 140045696608000 [Note] WSREP: Received NON-PRIMARY.
2017-10-21 21:57:36 140046341503744 [Note] WSREP: New cluster view: global state: 746e46f3-8f76-11e7-9c1c-a220eabab4ba:20626756, view# -1: non-Primary, number of nodes: 1, my index: 0, protocol version -1
2017-10-21 21:57:36 140046341503744 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.
2017-10-21 21:57:37 140045705000704 [Warning] WSREP: last inactive check more than PT1.5S ago (PT3.50413S), skipping check
2017-10-21 21:57:39 140045705000704 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') connection established to 26d54452 tcp://10.62.154.13:4567
2017-10-21 21:57:39 140045705000704 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') turning message relay requesting on, nonlive peers:
2017-10-21 21:57:40 140045705000704 [Note] WSREP: declaring 26d54452 at tcp://10.62.154.13:4567 stable
2017-10-21 21:57:40 140045705000704 [Warning] WSREP: no nodes coming from prim view, prim not possible
2017-10-21 21:57:40 140045705000704 [Note] WSREP: view(view_id(NON_PRIM,069ddd4c,110) memb {
	069ddd4c,0
	26d54452,0
} joined {
} left {
} partitioned {
})
2017-10-21 21:57:40 140045696608000 [Note] WSREP: New COMPONENT: primary = no, bootstrap = no, my_idx = 0, memb_num = 2
2017-10-21 21:57:40 140045696608000 [Note] WSREP: Flow-control interval: [23, 23]
2017-10-21 21:57:40 140045696608000 [Note] WSREP: Received NON-PRIMARY.
2017-10-21 21:57:40 140046341503744 [Note] WSREP: New cluster view: global state: 746e46f3-8f76-11e7-9c1c-a220eabab4ba:20626756, view# -1: non-Primary, number of nodes: 2, my index: 0, protocol version -1
2017-10-21 21:57:40 140046341503744 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.
2017-10-21 21:57:43 140045705000704 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') turning message relay requesting off
2017-10-21 21:57:52 140045705000704 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') connection established to 11247975 tcp://10.62.154.4:4567
2017-10-21 21:57:52 140045705000704 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') turning message relay requesting on, nonlive peers:
2017-10-21 21:57:52 140045705000704 [Note] WSREP: declaring 11247975 at tcp://10.62.154.4:4567 stable
2017-10-21 21:57:52 140045705000704 [Note] WSREP: declaring 26d54452 at tcp://10.62.154.13:4567 stable
2017-10-21 21:57:52 140045705000704 [Warning] WSREP: no nodes coming from prim view, prim not possible
2017-10-21 21:57:52 140045705000704 [Note] WSREP: view(view_id(NON_PRIM,069ddd4c,111) memb {
	069ddd4c,0
	11247975,0
	26d54452,0
} joined {
} left {
} partitioned {
})
2017-10-21 21:57:52 140045705000704 [Warning] WSREP: node uuid: 11247975 last_prim(type: 2, uuid: 11247975) is inconsistent to restored view(type: V_NON_PRIM, uuid: 069ddd4c
2017-10-21 21:57:52 140045696608000 [Note] WSREP: New COMPONENT: primary = no, bootstrap = no, my_idx = 0, memb_num = 3
2017-10-21 21:57:52 140045696608000 [Note] WSREP: Flow-control interval: [28, 28]
2017-10-21 21:57:52 140045696608000 [Note] WSREP: Received NON-PRIMARY.
2017-10-21 21:57:52 140046341503744 [Note] WSREP: New cluster view: global state: 746e46f3-8f76-11e7-9c1c-a220eabab4ba:20626756, view# -1: non-Primary, number of nodes: 3, my index: 0, protocol version -1
2017-10-21 21:57:52 140046341503744 [Note] WSREP: wsrep_notify_cmd is not defined, skipping notification.
2017-10-21 21:57:55 140045705000704 [Note] WSREP: (069ddd4c, 'tcp://0.0.0.0:4567') turning message relay requesting off
```

## 장애 원인
----
아래 장애 원인은 Galera Cluster 복구 메커니즘을 알지 못한 상태에서 생각한 것임.

* IaaS 작업 내용에 비추어 보면 VM 이 정상인 상태에서 Network 만 장시간 단절되었을 수 있음.
* 각 노드가 Network 단절로 인해 고립된 형태로 파티셔닝 되었을 수 있음.
* 이후 Network 복원 시점에 클러스터링 복원을 위해 서비스 포트(3306)를 일시적으로 내렸을 가능성 있음
* 클러스터 복원이 원활히 이루어지지 않고 deadlock 되면서 여타의 복원 프로세스가 연쇄적으로 동작하지 않게 됨.

## 작업 내역
----
장애 VM 중 하나를 재기동 하기 위해 종료: drain script timeout 으로 실패함.
```
# bosh -d mysql stop mysql/fb2639f6-23dc-4b19-afaf-a9dce3df4dc7
```

deployment 전체를 재기동: 서비스 정상화 됨 (bootstrap errend job 을 실행시키지 않아도 되는 점이 특이함)
```
# bosh -d mysql restart
```
