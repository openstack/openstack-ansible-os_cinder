OpenStack-Ansible Cinder
########################

This Ansible role installs and configures OpenStack Cinder.

The following Cinder services are managed by the role:
    * cinder-api
    * cinder-volume
    * cinder-scheduler

By default, Cinder API v1 and v2 are both enabled.

Support for various Cinder backends is supported by the role. See role
internals for further details.

Support for volume backups to Swift or Ceph is support by the role. See role
internals for further details.

Default Variables
=================

.. literalinclude:: ../../defaults/main.yml
   :language: yaml
   :start-after: under the License.


Required Variables
==================

This list is not exhaustive at present. See role internals for further
details.

.. code-block:: yaml

      # Comma separated list of Glance API servers
      cinder_glance_api_servers: "http://glance_server:9292"

      # Hostname or IP address of the Galera database
      cinder_galera_address: "1.2.3.4"

Example Playbook
================

.. code-block:: yaml

    - name: Installation and setup of cinder
      hosts: cinder_all
      user: root
      roles:
        - { role: "os_cinder", tags: [ "os-cinder" ] }
      vars:
        cinder_glance_api_servers: "http://glance_server:9292"
        cinder_galera_address: "{{ internal_lb_vip_address }}"
