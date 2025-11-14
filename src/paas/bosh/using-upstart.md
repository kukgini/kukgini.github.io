# Using upstart on ubuntu

오래 돌아야 하는 서버 또는 워커를 관리하는 스마트하지 않은 방법
-----------------------------------------------

* screen 이나 tmux 안에 띄워놓고 잊어버리기
* nohup 으로 실행해두고 잊어버리기

스마트하지 않은 방법의 문제점
-----------------------------------------------

* 프로세스가 꺼졌는지 한참동안 모르고 있다가 당황하기
* 시스템 재부팅 될때 두려움에 떨기

우분투에서 기본으로 제공되는 upstart 를 사용하는 스마트한 방법
-----------------------------------------------

* 시스템 부팅 시에 서비스 띄우기
* 다른 서비스가 시작된 후에 서비스 띄우기
* 프로세스가 오류로 꺼지면 자동으로 다시 띄우기
* stdout/stderr 를 로그 파일에 기록하기
* 로그 파일이 커지면 쪼개기

## 설정 파일 설치하기

upstart 서비스 설정 파일은 /etc/init 에 모여 있다. 따라서 /etc/init 디렉토리에 <서비스명>.conf 파일을 만들어 넣으면 된다.

## 심볼릭 링크로 설치하기

/etc/init 에는 시스템 서비스의 설정 파일도 모두 들어있으므로 관리의 용이성을 위해 별도의 디렉터리에 서비스 설정 파일을 분리해 보관하는 것이 권장됨. 분리된 설정은 심볼록 링크를 걸어서 같은 효과를 누릴 수 있다.
```
sudo ln -s /home/my/service/some.conf /etc/init/
```
**Caution!!** /etc/init/ 에 직접 파일을 만들었을 때와 달리 변경사항을 upstart 가 감지하지 못하므로 다음 명령을 통해 명시적인 재로딩을 수행해야 함.
```
sudo initctl reload-configuration
```

## 서비스 관리

* sudo start <service_name>
* sudo stop <service_name>
* sudo restart <service_name> (주의: 설정 파일을 다시 읽어오지 않음. 설정 변경시 stop/start 혹은 reload 할 것)
* sudo reload <service_name> (프로세스에 HUP 시그널을 보내는 것임

## 설정 파일 작성

### 명령어 지정

가장 간단한 방법은 exec 뒤에 명령얼르 쓰면 됨.
```
exec uname -a
```
긴 쉘 스크립트가 필요한 경우 script 구문을 쓰면 됨.
```
script
    sleep 5
        uname -a
end script
```
어떤 명령어들은 실행하면 자동으로 백그라운드로 돌아가는 (detach/daemonize) 경우가 있는데 이를 방지하는 옵션이 있다면 켜두는 것이 좋다 (보통 --foreground 혹은 --nodetach)

Foregroup mode 실행 옵션이 없는 경우 [여기](http://upstart.ubuntu.com/cookbook/#expect) 를 참고 해서 설정하시라.

### 자동 시작 / 중단 조건

서비스를 조건에 따라 시작되거나 중단되게 할 수 있다. 가장 흔히 사용하는 조건은 부팅시 시작, 종료시 중단일텐데 다음과 같이 적으면 됨.
```
start on runlevel [2345]
stop on runlevel [016]
```
만약, 특정 서비스가 시작된 이후에 띄우고 싶다면 다음과 같이 아라
```
start on started <other_service>
```
and 연산자로 여러 조건을 조합할 수도 있다.
```
start on started mysql and started nginx
```

### 실행 권한 설정

프로세스는 기본적으로 root 권한을 가지고 실행된다. 보안을 강화하기 위해 별도의 사용자/그룹 권한으로 실행하고자 할 경우 다음과 같이 한다.
```
setuid <username>
setgid <groupname>
```

### 실행 환경 설정

환경변수를 설정 (env KEY=vlaue) 한 경우에는 exec/script 구문 내에서 $KEY 로 참조할 수 있다.

디렉터리 변경은 다음과 같이 할 수 있다.
```
chdir /path/to/current/dir
```

### 자동 재시작 (respawn)

프로세스가 예기치 않게 종료 되었을 때 자동으로 재실행 하는 방법
```
respawn
```
프로세스가 너무 빨리 되살아나는 것을 방지하기 위해 5초 동안 10번 재시작이 되면 더이상 재시작 하지 않는다. 이 제한은 respawn limit 으로 바꿀수 있음

**Caution!!** respawn limit 이 설정되었더라도 respawn 구문이 없으면 자동 재시작이 되지 않는다.

```
respawn
respawn limit 5 10 # 10 초동안 5 번 재시작이 되면 포기한다.
respawn limit unlimited
```

### 로그 파일

stdout/stderr 로 출력된 내용은 /var/log/upstart/<service_name>.log 에 저장된다. 우분투 (14.4) 기준으로는 이 로그 파일이 하루에 한번 분할 되고 최대 7개의 파일이 유지되는 것이 기분 설정이다. /etc/logrotate.d/upstart 에서 설정을 바꿀 수 있다.

### 설정 파일 예제

gunicorn 으로 파이선 웹 어플리케이션을 실행하는 예제
```
start on runlevel [2345]
stop on runlevel [016]

respawn
setuid www-data
setgid www-data
env PORT=11000
chdir /home/ditto/animeta

exec env/bin/gunicorn animeta.wsgi --bind 127.0.0.1:$PORT --error-logfile -
```

## Alternatives

1. 우분투 차기 버전에서 upstart 가 [systemd](http://www.freedesktop.org/wiki/Software/systemd/) 로 대체될 예정이라고 함. 이미 Debian linux 에서는 systemd 가 기본으로 탑재되어 있다.
1. 시스템과 독립적으로 작동하면서 upstart 같은 기능을 제공하는 [supervisor](http://supervisord.org/) 라는 것도 많이 쓰이고 있음

## 참조 자료

* http://blog.sapzil.org/2014/08/12/upstart/
