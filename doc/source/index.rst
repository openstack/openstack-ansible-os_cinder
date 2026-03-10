=================================
Cinder role for OpenStack-Ansible
=================================

This Ansible role installs and configures OpenStack cinder.

The following cinder services are managed by the role:
    * cinder-api
    * cinder-volume
    * cinder-backup
    * cinder-scheduler

Only Cinder API v3 is enabled at the moment by default

.. toctree::
   :maxdepth: 2

   configure-cinder.rst
   deployment-options.rst

To clone or view the source code for this repository, visit the role repository
for `os_cinder <https://github.com/openstack/openstack-ansible-os_cinder>`_.

Default variables
~~~~~~~~~~~~~~~~~

.. literalinclude:: ../../defaults/main.yml
   :language: yaml
   :start-after: under the License.


Dependencies
~~~~~~~~~~~~

This role needs pip >= 7.1 installed on the target host.

This role needs to have the following variables defined:

.. code-block:: yaml

      # Comma separated list of Glance API servers
      cinder_glance_api_servers: "http://glance_server:9292"

      # Hostname or IP address of the Galera database
      cinder_galera_address: "1.2.3.4"

This list is not exhaustive at present. See role internals for further
details.

Example playbook
~~~~~~~~~~~~~~~~

.. literalinclude:: ../../examples/playbook.yml
   :language: yaml

Tags
~~~~

This role supports series of common tags, like ``ceph``, ``common-db``,
``common-db``, ``common-mq``, ``post-install`` as well as role-specific
tags like: ``cinder-install`` and ``cinder-config``.

The ``cinder-install`` tag can be used to install and upgrade.

The ``cinder-config`` tag can be used to maintain configuration of the
service.
