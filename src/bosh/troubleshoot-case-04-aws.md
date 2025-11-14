# Case 4 AWS

## Goal

deploy cf with cf-deployment 0.3.0 on AWS

## Problem

```
	consul:/var/vcap/sys/log/consul_agent# tail -f consul_agent.stderr.log
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
	error during start: timeout exceeded: "log not in sync"
```

## Solution

i experienced a similar error recently. if you look at some similar situations that pivotal cloud foundry customers have encountered. talking with the engineers that work on cloud foundry and consul, they recommend:

stopping the monit job in each consul VM monit stop consul_agent
removing all the consul data under rm -rf /var/vcap/store/consul_agent/*
after all VMs cleared of consul agent data, starting monit in each one
then it all came back for me.

## Reference

- https://github.com/cloudfoundry/bosh/issues/1227