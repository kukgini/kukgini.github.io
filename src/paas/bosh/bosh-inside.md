# BOSH Inside

## BOSH Architecture

(다이어그램)

.. _ImageLink: 
![BOSH Architecture](https://image.slidesharecdn.com/cloudfoundry-integration-with-openstack-and-docker-bangalorecfmeetup-150402095116-conversion-gate01/95/cloud-foundry-integrationwithopenstackanddockerbangalorecfmeetup-17-638.jpg?cb=1427968572 "BOSH Architecture")

## BOSH Release 의 구성 요소

* nats
* postgres (table schema, initial data)
* blobstore (structure, size)
* director
* hm9000
* ntp
* agnet (nats?)

## BOSH Release 의 추가 요소

* Config Server
* Turbulence
* local-dns
* powerdns
* UAA Integration

## Security 를 위한 설정


## Jumpbox

이건 뭘까? 최근에 생긴건데.

## bosh-lite 의 구성

![bosh-lite](https://i.stack.imgur.com/z3HrU.png)

https://stackoverflow.com/questions/41303870/how-can-i-deploy-a-bosh-director-on-bosh-lite

The following is an extension of this question: https://cloudfoundry.slack.com/archives/C0SBBBJSZ/p1496276319629886

After checking the following logs in the generated bosh vm, it seems that there is not enough disk space.

```
	2017-06-07_06: 36: 14.10298 [main] 2017/06/07 06:36:14 ERROR - App setup Running bootstrap: Setting up ephemeral disk: Creating ephemeral partitions on root device: Insufficient remaining disk space (150225408B) for ephemeral Partition (min: 1073741824B)
```

I tried to modify deployment menifest like this:

```
	.../bosh-deployment $ git diff bosh.yml
	Diff --git a / bosh.yml b / bosh.yml
	Index 14890d8..dfe4e93 100644
	--- a / bosh.yml
	+++ b / bosh.yml
	@@ -16,7 +16,7 @@ resource_pools:

	Disk_pools:
	- name: disks
	- disk_size: 32_768
	+ disk_size: 1_000_000

	Networks:
	- name: default
	```
	```$ git diff openstack/cpi.yml
	diff --git a/openstack/cpi.yml b/openstack/cpi.yml
	index 121bc01..15cd9f4 100644
	--- a/openstack/cpi.yml
	+++ b/openstack/cpi.yml
	@@ -10,14 +10,15 @@
	# Configure sizes
	- type: replace
	path: /resource_pools/name=vms/cloud_properties?
	value:
	- instance_type: m1.xlarge
	+ instance_type: linux_c8m64
	+ ephemeral_disk: {size: 1_000_000, type: <my-cinder-type>}
	availability_zone: ((az))
```

But it did not working. bosh/0 vm still have only 3GB.

```
	/:~$ df -h
	Filesystem Size Used Avail Use% Mounted on
	udev 7.9G 4.0K 7.9G 1% /dev
	tmpfs 1.6G 296K 1.6G 1% /run
	/dev/vda1 2.8G 1.2G 1.5G 43% /
	none 4.0K 0 4.0K 0% /sys/fs/cgroup
	none 5.0M 0 5.0M 0% /run/lock
	none 7.9G 0 7.9G 0% /run/shm
	none 100M 0 100M 0% /run/user
```

QuestionIs: How to increase root volume or add ephemeral disk?