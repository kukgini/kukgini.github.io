BOSH2 CLI 로 VM 에 로그인 하기
================================================================================
.. code-block:: bash

   bosh -e <alias-name> -d <deployment-name> ssh api/... --opts=-v --gw-private-key ./director.key
