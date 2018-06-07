# BOSH lite resizing

## resizing memory

최초 시작시 (아직 VM Provisioning 이 수행되기 전) Vagrantfile 을 열어 config.vm.provider 내에 v.memory 를 MB 단위로 지정하고 vagrant up 을 하면 메모리가 조정된다.

```
Vagrant.configure('2') do |config|
  config.vm.box = 'cloudfoundry/bosh-lite'

  config.vm.provider :virtualbox do |v, override|
    override.vm.box_version = '9000.137.0' # ci:replace
    # To use a different IP address for the bosh-lite director, uncomment this line:
    # override.vm.network :private_network, ip: '192.168.59.4', id: :local
    v.memory = 40960
  end

  config.vm.provider :aws do |v, override|
    override.vm.box_version = '9000.137.0' # ci:replace
    # To turn off public IP echoing, uncomment this line:
    # override.vm.provision :shell, id: "public_ip", run: "always", inline: "/bin/true"

    # To turn off CF port forwarding, uncomment this line:
    # override.vm.provision :shell, id: "port_forwarding", run: "always", inline: "/bin/true"

    # Following minimal config is for Vagrant 1.7 since it loads this file before downloading the box.
    # (Must not fail due to missing ENV variables because this file is loaded for all providers)
    v.access_key_id = ENV['BOSH_AWS_ACCESS_KEY_ID'] || ''
    v.secret_access_key = ENV['BOSH_AWS_SECRET_ACCESS_KEY'] || ''
    v.ami = ''
  end

  config.vm.provider :vmware_fusion do |v, override|
    override.vm.box = 'cloudfoundry/no-support-for-bosh-lite-on-fusion'
    #we no longer build current boxes for vmware_fusion
    #ensure that this fails. otherwise the user gets an old box
  end

  config.vm.provider :vmware_workstation do |v, override|
    override.vm.box = 'cloudfoundry/no-support-for-bosh-lite-on-workstation'
    #we no longer build current boxes for vmware_workstation
    #ensure that this fails. otherwise the user gets an old box
  end
end
```

## resizing disk

디스크의 조정은 다음 문서를 참조했다.
https://medium.com/@phirschybar/resize-your-vagrant-virtualbox-disk-3c0fbc607817

Vagrant 가 Provisioning 한 디스크는 vmdk 포맷인데 이 포멧은 VBoxManager 를 통해 동적으로 resizing 을 할 수 없다. 따라서 다음 과정을 통해 용량 증설이 가능하다.

* vmdk -> vdi 로 포맷 변경
* vdi resizing
* guest 에서 파티션 생성
* LVM 으로 새로 생성된 파티션을 기존 파티션에 붙이기

```
~/host> vagrant halt
~/host> cd /path/where/your/vbox/.vmdk/files/are
~/host> VBoxManage clonehd my.vmdk out.vdi --format VDI
~/host> VBoxManage modifyhd out.vdi --resize 51200 
```
이렇게 하면 이미 떠있던 VM 이 halt 된다. 나중에 다시 VM 을 기동하여도 warden 컨테이너는 제대로 기동이 안되므로 deploy 를 수행하기 전에 용량 증설을 먼저 해두어야 한다.

그런 다음 VirtualBox GUI 를 통해 하드 드라이브를 새로 만든 .vdi 이미지로 바꿔치기 한다.
(select VM > settings > storage > remove existing hard drive and add bew hard drive)

```
~/host> vagrant up
~/host> vagrant ssh
~/guest> sudo cfdisk /dev/sda
```
이렇게 하면 VM 이 기동되고 파티션 정보를 보여주는 대화형 창이 뜬다. 새로운 Space 를 선택후 다음과 같은 명령을 수행한다.
(Select new space available > select "NEW" > select "Primary" > select "Write")
/sda3 가 새로운 파티션으로 나타날 것이다.

``` 
~/guest> exit
~/host> vagrant reload
~/host> vagrant ssh
~/guest> sudo pvcreate /dev/sda3
```
만약 pvcreate 명령이 설치되어 있지 않다면 **apt install lvm2** 로 설치힌다.

```
~/guest> sudo pvdisplay | grep "VG Name"
```
이것은 볼륨 그룹 이름을 출력해 줄 것이다. 내 경우는 ubuntu-vg 였다. 다음과 같이 하여 볼륨 그룹에 새로 생성된 파티션을 붙여 확장한다. VG-NAME 을 자신의 것으로 바꿀것.

```
~/guest> sudo vgextend VG-NAME /dev/sda3
~/guest> sudo lvextend /dev/VG-NAME/root /dev/sda3
~/guest> sudo resize2fs /dev/VG-NAME/root
~/guest> df -h
``` 

마지막 명령은 새롭게 조정된 디스크 볼륨을 보여줄 것이다.