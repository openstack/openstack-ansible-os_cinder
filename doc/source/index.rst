os_cinder Role Docs
===================

The os_cinder role is used to to deploy, configure and install OpenStack Block
Storage.

This role will install the following:
    * cinder-api
    * cinder-volume
    * cinder-scheduler

Basic Role Example
^^^^^^^^^^^^^^^^^^

.. code-block:: yaml

    - name: Installation and setup of cinder
      hosts: cinder_all
      user: root
      roles:
        - { role: "os_cinder", tags: [ "os-cinder" ] }
      vars:
        cinder_galera_address: "{{ internal_lb_vip_address }}"
