# OpenStack Case 1

## Goal

I'm tring to deploy bosh director using bosh cli v2 (2.0.16) on Openstack(mikata) from external network. 
This guide (https://bosh.io/docs/init-external-ip.html) says external_ip variable can do this. 

## Problem

But, actually I got this :

```
    Starting registry... Finished (00:00:00)
    Uploading stemcell 'bosh-openstack-kvm-ubuntu-trusty-go_agent-raw/3363.20'... Finished (00:00:49)

    Started deploying
    Deleting VM 'cd5b0f1c-8a33-494a-b539-df3c646af8d6'... Finished (00:00:00)
    Creating VM for instance 'bosh/0' from stemcell 'df4a4001-3334-4d28-8172-6966033250d1'... Finished (00:00:58)
    Waiting for the agent on VM '1c3dc920-09d7-4718-89ac-520aa0689e18' to be ready... Failed (00:08:30)
    Failed deploying (00:09:30)

    Stopping registry... Finished (00:00:00)
    Cleaning up rendered CPI jobs... Finished (00:00:00)

    Deploying:
    Creating instance 'bosh/0':
        Waiting until instance is ready:
        Starting SSH tunnel:
            Starting SSH tunnel:
            Failed to connect to remote server:
                dial tcp xx.xx.xx.xx:22: getsockopt: connection timed out
```

xx.xx.xx.xx is internal ip not external. What went wrong? 

## Solution

It's because I missed operation : external-ip-with-registry-not-recommended.yml

'-not-recommended' means bosh develepment team don't want to use this operation. Instead, encourage to use sandbox machine.

