# Custom User Provided Service (CUPS)

CloudFoundry Marketplace 를 통해서 제공되는 서비스가 아닌 경우 마치 사용자가 스스로 서비스인 것 처럼 등록하여 쓸 수 있는 기능.

## Binding Credentials

## Eureka Server CUPS 로 등록하기

우선 다음 eureka 서버 샘플을 내려 받는다: https://github.com/making/cf-eureka-server

내 경우는 이녀석이 너무 만들어 진지 오래 되어 SpringBoot 버전을 상향하였다. (1.1.3 -> 1.3.3)

build 한 후 manifest 를 작성하여 push 한다. push 가 되면서 Router url 이 발급되었을 것이다.

다음 명령으로 서비스화 한다.

```
cf uups eureka -p '{"name":"eureka", "uri":"http://eureka:changeme@<router-url-of-pushed-app>"}'
```