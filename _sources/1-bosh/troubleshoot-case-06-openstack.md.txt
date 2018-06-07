# Trouble Shoot case 6 OpenStack

- bosh cli Ver 2.0.16
- bosh-deployment Ver 

include({{ch-02-02.files/cmd.log}})

.. literalinclude:: ch-02-02.files/res.log

.. literalinclude:: ch-02-02.files/manual_ssh.log

KeyPair 가 손상 되었을 가능성은 다음 명령으로 확인 가능하다:

.. literalinclude:: ch-02-02.files/validate_key.log

이렇게 출력한 2개의 값이 서로 같으면 public/private KeyPair 는 서로 맞는 짝이다.

검색 키워드: openstack unable to ssh keypair

정확한 원인은 IaaS 운영자가 알려주지 않았으나 Provisioning 도구를 통해 수동으로 생성한 VM 들의 부팅 과정에서
meta-data 서버에 접근이 안되는 로그가 찍히는 것으로 보아 meta-data 서버에 장애가 발생한 듯 보임.

참고자료:

- https://lists.cloudfoundry.org/archives/list/cf-bosh@lists.cloudfoundry.org/thread/TVMRTEMK2TS6JMPTHFFSIWAU6AULXPKT/
- https://bosh.io/docs/openstack-cpi.html#errors
- https://ask.openstack.org/en/question/52898/unable-to-ssh-instance-with-keypair/
