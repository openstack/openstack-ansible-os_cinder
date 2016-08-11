========================
OpenStack-Ansible cinder
========================

This Ansible role installs and configures OpenStack Cinder.

The following Cinder services are managed by the role:
    * cinder-api
    * cinder-volume
    * cinder-scheduler

By default, Cinder API v1 and v2 are both enabled.

Support for various Cinder backends is supported by the role. See role
internals for further details.

Support for volume backups to Swift or Ceph is supported by the role. See role
internals for further details.
