Cloud Native Application
================================================================================
Java Application 의 경우 다음과 같은 형식으로 프로세스가 기동된다.

.. code-block:: bash

    /home/vcap/app/.java-buildpack/open_jdk_jre/bin/java 
        -Djava.io.tmpdir=/home/vcap/tmp 
        -XX:OnOutOfMemoryError=/home/vcap/app/.java-buildpack/open_jdk_jre/bin/killjava.sh 
        -XX:MaxMetaspaceSize=64M -Xss228K -Xms317161K -XX:MetaspaceSize=64M -Xmx317161K 
        -Djavax.net.ssl.trustStore=/home/vcap/app/.java-buildpack/container_certificate_trust_store/truststore.jks 
        -Djavax.net.ssl.trustStorePassword=java-buildpack-trust-store-password 
        -cp /home/vcap/app/.:/home/vcap/app/* 
        <java-main-class-which-has-static-main-function>

#. JDK/JRE 는 /home/vcap/app/.java-buildpack/open_jdk_jre 에 설치된다.
#. $TEMP dir 는 /home/vcap/tmp 이므로 반드시 java.io.tmpdir 환경 변수를 참조해야 함
#. OutOfMemoryError 가 발생할 경우 killjava.sh 가 실행된다.
#. 메모리 크기는 컨테이너에 할당된 양에 맞추어 동적으로 조정된다.
#. truststore.jks 라 customize 될 수 있다.
#. jar 의 내용이 압축이 풀린 상태로 실행된다.
