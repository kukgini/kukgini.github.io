HowTo Simply TcpDump and get.
================================================================================

Cloud Controller 와 Gorouter 간 SSL 통신 실패 원인 분석을 위해 TcpDump 를 수행할 때 사용되었음.

.. code-block:: bash

  FILE=pcap-5-router.pcap IF=`ifconfig -s | grep "^wc" | awk '{ print $1}'` eval 'sudo ./tcpdump -i $IF -w $FILE'
  FILE=pcap-5-api.pcap IF=`ifconfig -s | grep "^wc" | awk '{ print $1}'` eval 'sudo ./tcpdump -i $IF -w $FILE'
  FILE=pcap-5-haproxy.pcap IF=`ifconfig -s | grep "^wc" | awk '{ print $1}'` eval 'sudo ./tcpdump -i $IF -w $FILE'

위의 방식과 유사 하지만 bosh-lite 에서 director 는 다른 vm 과 달리 vagrant 내에서 수행하며 NIC name 이 vboxnet0 이다.

.. code-block:: bash

  FILE=pcap-5-cli.pcap IF=vboxnet0 eval 'sudo tcpdump -i $IF -w $FILE'

  cf create-service-broker p-mysql admin password https://p-mysql.bosh-lite.com:443

캡쳐가 끝나면 덤프 데이터를 내 PC 로 가져와야 하는데 원격 로그인을 위한 Public Key 를 생성한 후
ssh 로 전송한다.

.. code-block:: bash

  cat << EOL > ~/.ssh/id_rsa
  -----BEGIN RSA PRIVATE KEY-----
  MIIEpQIBAAKCAQEAqCXkcoUt4aVnfVw1ToLUZA6OPFZ25AFC5m+ck+7PgtTc7jIn
  .....
  CBxlZtak/RURKlNT0E+K1mVK9YwJPYZ8pt7jtszjjYKaIh+EZEl6YDo=
  -----END RSA PRIVATE KEY-----
  EOL
  chmod go-rwX ~/.ssh/id_rsa
  scp *.pcap <target_server>:~/
