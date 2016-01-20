GridFTP SSH
===========

Install Globus GridFTP for use over SSH (`sshftp://`).

- http://toolkit.globus.org/toolkit/docs/latest-stable/admin/install/

Only the control channel is transported over SSH, data is transferred separately.
It is not necessary to run the server as a daemon, it will be started automatically by the client.

Requirements
------------

This role will also install OpenSSH client and server packages.

Author Information
------------------

ome-devel@lists.openmicroscopy.org.uk
