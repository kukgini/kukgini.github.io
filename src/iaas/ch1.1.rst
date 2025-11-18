Chapter 1 Command-Line Interface (CLI)
================================================================================

Login
-------------------------------------------

.. code-block:: bash

    #!/bin/bash
    openstack --os-auth-url http://xx.xx.xx.xx:5000/v3 \
            --os-identity-api-version 3 \
            --os-auth-type v3password \
            --os-project-domain-id <project-domain-id> \
            --os-project-name <project_name> \
            --os-user-domain-id <user-domain-id> \
            --os-username <username> \
            --os-password <password>

Floating IP 할당
-------------------------------------------

.. code-block:: text

    (openstack) server list
    (openstack) floating ip list
    (openstack) server add floating ip <server-id> <floating-ip-addr>

Uncategorized
================================================================================

- [Official Documentation](http://docs.openstack.org/index.html)
- [Newton releases 의 구성요소](https://releases.openstack.org/newton/index.html)
- [Installation Guides](http://docs.openstack.org/index.html#install-guides)


