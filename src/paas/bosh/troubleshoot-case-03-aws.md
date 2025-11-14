# Case 3 AWS

## Goal

ssh log in into VM which is bosh created.

## Problem
```
    $ bosh -e aws-us -d cf ssh diego-cell/fb512f76-xxxx
    Using environment 'https://xx.xx.xx.xx:25555' as client 'admin'

    Using deployment 'cf'

    Task 324. Done
    Warning: Permanently added 'xx.xx.xx.xx' (ECDSA) to the list of known hosts.
    Unauthorized use is strictly prohibited. All access and activity
    is subject to logging and monitoring.
    Permission denied (publickey).
    ssh_exchange_identification: Connection closed by remote host

    Running SSH:
    1 error(s) occurred:

    * Running command: 'ssh -tt -o ServerAliveInterval=30 -o ForwardAgent=no -o PasswordAuthentication=no -o IdentitiesOnly=yes -o IdentityFile=/home/.../.bosh/tmp/ssh-priv-key263879026 -o StrictHostKeyChecking=yes -o UserKnownHostsFile=/home/.../.bosh/tmp/ssh-known-hosts663697449 -o ProxyCommand=ssh -tt -W %!h(MISSING):%!p(MISSING) -l vcap xx.xx.xx.xx -o ServerAliveInterval=30 -o ForwardAgent=no -o ClearAllForwardings=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null 10.0.16.14 -l bosh_a1da6765d1fc4866', stdout: '', stderr: '': exit status 255

    Exit code 1
```

## Solution

Specifying a private key for the `bosh ssh` session via the `--gw-private-key` flag or the equivalent `BOSH_GW_PRIVATE_KEY` environment variable.

If you created bosh environment using bbl, You can run `eval "$(bbl print-env)"` to get gateway and private key set, that should fix your `bosh ssh`

```
    $ bbl ssh-key
    -----BEGIN RSA PRIVATE KEY-----
    MIIEowIBAAKCAQEA73eXEMQMAqgsYWSNcqPTZVPNIscX3Gljf2CPs9C5mI0KaV3ePXi+alPY6o3P
    ...
```

## References

- https://github.com/cloudfoundry/bosh-cli/issues/128
- https://cloudfoundry.slack.com/archives/C2DBC3YGZ/p1494894883740994