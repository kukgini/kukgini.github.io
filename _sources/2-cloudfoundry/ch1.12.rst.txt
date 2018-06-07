bosh-init 시 Open Stack Nova Bug 관련
================================================================================
bosh-init 으로 director vm 생성시 다음과 같은 오류가 발생한다면

.. code-block:: none

  asn1: structure error: superfluous leading zeros in length

오픈 스택 Nova 버그에 기인한 것으로 생각된다.

관련 issue: https://bugs.launchpad.net/nova/+bug/1483132

.. literalinclude:: files/openstack-nova-bug.log

