# OpenStack Case 2

## Goal

I'm tring to deploy bosh director using bosh cli v2 (2.0.16) on Openstack(mikata) from external network.

## Problem

```
    $ ./bosh create-env bosh.yml ....
    Deployment manifest: '/home/.../bosh-deployment/bosh.yml'
    Deployment state: '/home/.../bosh-1/state.json'

    Started validating
    Downloading release 'bosh'... Skipped [Found in local cache] (00:00:00)
    Validating release 'bosh'... Finished (00:00:00)
    Downloading release 'bosh-openstack-cpi'... Skipped [Found in local cache] (00:00:00)
    Validating release 'bosh-openstack-cpi'... Finished (00:00:00)
    Validating cpi release... Finished (00:00:00)
    Validating deployment manifest... Finished (00:00:00)
    Downloading stemcell... Skipped [Found in local cache] (00:00:00)
    Validating stemcell... Finished (00:00:02)
    Finished validating (00:00:03)

    Started installing CPI
    Compiling package 'ruby_openstack_cpi/9485b575...'... Finished (00:00:00)
    Compiling package 'bosh_openstack_cpi/dd0bab98...'... Finished (00:00:00)
    Installing packages... Finished (00:00:00)
    Rendering job templates... Finished (00:00:00)
    Installing job 'openstack_cpi'... Finished (00:00:00)
    Finished installing CPI (00:00:00)

    Starting registry... Finished (00:00:00)
    Uploading stemcell 'bosh-openstack-kvm-ubuntu-trusty-go_agent-raw/3363.20'... Skipped [Stemcell already uploaded] (00:00:00)

    Started deploying
    Waiting for the agent on VM '011c4fb3-xxxx'... Failed (00:00:12)
    Deleting VM '011c4fb3-xxxx'... Finished (00:00:07)
    Creating VM for instance 'bosh/0' from stemcell '88e02896-xxxx'... Finished (00:00:37)
    Waiting for the agent on VM '6fdfc19a-xxxx' to be ready...


    Failed (00:10:08)
    Failed deploying (00:11:06)

    Stopping registry... Finished (00:00:00)
    Cleaning up rendered CPI jobs... Finished (00:00:00)

    Deploying:
    Creating instance 'bosh/0':
        Waiting until instance is ready:
        Sending ping to the agent:
            Performing request to agent endpoint 'https://mbus:xxxx@xx.xx.xx.xx:6868/agent':
            Performing POST request:
                Post https://mbus:<redacted>@xx.xx.xx.xx:6868/agent: dial tcp xx.xx.xx.xx:6868: getsockopt: connection refused

    Exit code 1
```

## Solution

NATS 서버가 Listening 하고 있는 6868 port 에 대한 접근 권한이 없는 것이 원인임. IaaS security group 설정에서 bosh security group 에 대해 6868 port 접근이 가능하도록 수정.

## References

none.