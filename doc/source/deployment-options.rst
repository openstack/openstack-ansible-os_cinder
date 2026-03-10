======================================
Deployment options for Cinder services
======================================

Running cinder-volume in containers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Historically, ``cinder-volume`` component is configured to run on metal
by default. As most of the backends for ``cinder-volume`` can run inside
of containers, it is recommended to spawn the service inside the container
whenever possible.

For that, create a file ``/etc/openstack_deploy/env.d/cinder.yml`` with the
following content:

.. code-block:: yaml

   container_skel:
     cinder_volumes_container:
       properties:
         is_metal: false


Spawning multiple cinder-volume containers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

While ``cinder-volume`` can handle multiple backends at the same time,
it might be not always preferable or desired due to multiple reasons.

Such reasons may be, but not limited by:

* Backends are using different networks, and it's preferable to isolate traffic
* Backends are having different capabilities, like support of active/active
  setup, so ``cinder-volume`` needs to be configured differently

In an example below we will describe on how to add an extra set of containers
to handle Ceph backend in active/active manner in addition to already existing
NFS backend.

Existing setup
--------------

Let's assume you have already running ``cinder-volume`` service with NFS
backend, running in an LXC container on a ``br-storage`` network.
For this use-case configuration may look like this:

* ``/etc/openstack_deploy/env.d/cinder.yml``

  .. code-block:: yaml

    container_skel:
      cinder_volumes_container:
        properties:
          is_metal: false

* ``/etc/openstack_deploy/env.d/nfs.yml``

  .. code-block:: yaml

    ---
    component_skel:
      nfs_server:
        belongs_to:
          - nfs_all

    container_skel:
      nfs_container:
        belongs_to:
          - nfs_block_containers
        contains:
          - nfs_server
        properties:
          is_metal: true

    physical_skel:
      nfs_block_containers:
        belongs_to:
          - all_containers
      nfs_block_hosts:
        belongs_to:
          - hosts

* ``/etc/openstack_deploy/openstack_user_config.yml``

  .. code-block:: yaml

    cidr_networks: &os_cidrs
      management: 172.29.236.0/22
      storage: 172.29.244.0/22

    global_overrides:
      provider_networks:
        - network:
          container_bridge: "br-storage"
          container_type: "veth"
          container_interface: "eth2"
          container_mtu: "9000"
          ip_from_q: "storage"
          address_prefix: "storage"
          type: "raw"
          group_binds:
            - cinder_volume
            - nova_compute
            - glance_all
    ....
    ....
    ....
    storage-infra_hosts: *controllers
    storage_hosts: *controllers
    nfs_block_hosts:
      nfs01:
        ip: 172.29.236.250

* ``/etc/openstack_deploy/group_vars/cinder_all.yml``

  .. code-block:: yaml

    cinder_backends:
      nfs:
        volume_backend_name: NFS
        volume_driver: cinder.volume.drivers.nfs.NfsDriver
        nfs_shares_config: /etc/cinder/nfs_shares
        nfs_qcow2_volumes: true
        nfs_mount_options: rw,nolock,noatime
        nfs_snapshot_support: true
        nas_secure_file_operations: false
        shares:
          - ip: nfs01
            share: /data/cinder-volume
        extra_volume_types:
          - unlimited

    cinder_qos_specs:
      - name: default
        options:
          consumer: front-end
          total_iops_sec_per_gb: 10
          total_iops_sec_per_gb_min: 300
          total_iops_sec_max: 10000
        cinder_volume_types:
          - nfs


Create and configure extra cinder-volume containers
---------------------------------------------------

Below we will describe on how to spawn another set of ``cinder-volume``
containers in Active/Active mode with Ceph backend on a different
network, while preserving NFS backend in its current state.

* Add new ``cinder_ceph_volumes`` group by bringing
  ``/etc/openstack_deploy/env.d/cinder.yml`` to the following shape:

  .. code-block:: yaml

    container_skel:
      cinder_volumes_container:
        properties:
          is_metal: false
      cinder_ceph_volumes_container:
        belongs_to:
          - storage_ceph_containers
        contains:
          - cinder_volume

    physical_skel:
      storage_ceph_containers:
        belongs_to:
          - all_containers
      storage_ceph_hosts:
        belongs_to:
          - hosts

* Add an extra network definition along with defining
  hosts for this newly created group to ``/etc/openstack_deploy/openstack_user_config.yml``.
  You also will need to re-bind an existing storage network to more specific
  group, as ``cinder-volume`` does include both NFS and Ceph containers.

  .. note::

    Existing storage network can be also used for new ``cinder-volume``. In
    this case you don't need to adjust ``group_binds`` for storage network
    or define another network.

  .. note::

    Defining ``coordination_hosts`` will result in Zookeeper cluster deployment
    into their own containers. Presence of coordination service is needed for
    proper Active/Active setup of ``cinder-volume`` service.

  .. code-block:: yaml

    cidr_networks: &os_cidrs
      management: 172.29.236.0/22
      storage: 172.29.244.0/22
      ceph_storage: 172.29.248.0/22

    global_overrides:
      provider_networks:
        - network:
          container_bridge: "br-storage"
          container_type: "veth"
          container_interface: "eth2"
          container_mtu: "9000"
          ip_from_q: "storage"
          address_prefix: "storage"
          type: "raw"
          group_binds:
            - cinder_volumes_container
            - nova_compute
            - glance_all
        - network:
          container_bridge: "br-ceph-storage"
          container_type: "veth"
          container_interface: "eth2"
          container_mtu: "9000"
          ip_from_q: "ceph_storage"
          address_prefix: "ceph_storage"
          type: "raw"
          group_binds:
            - cinder_ceph_volumes_container
            - nova_compute
    ....
    ....
    ....
    storage-infra_hosts: *controllers
    storage_hosts: *controllers
    storage_ceph_hosts: *controllers
    coordination_hosts: *controllers

* Add Ceph backend configuration for ``cinder-volume`` in
  ``/etc/openstack_deploy/group_vars/cinder_ceph_volumes_container.yml``

  .. code-block:: yaml

    ceph_cluster_name: ceph
    cinder_active_active_cluster: true
    cinder_active_active_cluster_name: "{{ ceph_cluster_name }}"
    cinder_backends:
      ceph:
        volume_backend_name: RBD
        volume_driver: cinder.volume.drivers.rbd.RBDDriver
        rbd_exclusive_cinder_pool: true
        rbd_flatten_volume_from_snapshot: true
        rbd_pool: volumes
        rbd_store_chunk_size: 8
        rbd_user: "{{ cinder_ceph_client }}"
        rbd_secret_uuid: "{{ cinder_ceph_client_uuid }}"
        rbd_cluster_name: "{{ ceph_cluster_name }}"
        rados_connect_timeout: -1
        rbd_max_clone_depth: 5
        extra_volume_types:
          - ceph-unlimited

* Move NFS backend configuration to a more specific group_vars file
  ``/etc/openstack_deploy/group_vars/cinder_volumes_container.yml``

  .. code-block:: yaml

    cinder_backends:
      nfs:
        volume_backend_name: NFS
        volume_driver: cinder.volume.drivers.nfs.NfsDriver
        nfs_shares_config: /etc/cinder/nfs_shares
        nfs_qcow2_volumes: true
        nfs_mount_options: rw,nolock,noatime
        nfs_snapshot_support: true
        nas_secure_file_operations: false
        shares:
          - ip: nfs01
            share: /data/cinder-volume
        extra_volume_types:
          - unlimited

* Extend QoS with new volume type for Ceph
  ``/etc/openstack_deploy/group_vars/cinder_all.yml``

  .. code-block:: yaml

    cinder_qos_specs:
      - name: default
        options:
          consumer: front-end
          total_iops_sec_per_gb: 10
          total_iops_sec_per_gb_min: 300
          total_iops_sec_max: 10000
        cinder_volume_types:
          - nfs
          - ceph

* Run the playbooks to spawn new containers and configure them accordingly

    .. code-block:: console

        openstack-ansible openstack.osa.containers_lxc_create --limit cinder_volumes,zookeeper_all,lxc_hosts
        openstack-ansible openstack.osa.cinder


Running cinder-backup separately from cinder-volume
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

At the moment, cinder-backup does not support multi-backend installations.
So in case of having multiple sets of ``cinder-volume`` containers for
different backends, you may want to consider separating ``cinder-backup`` from
``cinder-volume`` and spawn ``cinder-backup`` inside their own containers for
better transparency and control over configuration.


* Define ``env.d`` override for cinder_volume to remove cinder_backup and
  create a separate hosts group for them. So in
  ``/etc/openstack_deploy/env.d/cinder.yml``:

  .. code-block:: yaml

    ---
    container_skel:
      cinder_volumes_container:
        contains:
          - cinder_volume
        properties:
          is_metal: false
      cinder_backup_container:
        belongs_to:
          - storage_backup_containers
        contains:
          - cinder_backup

    physical_skel:
      storage_backup_containers:
        belongs_to:
          - all_containers
      storage_backup_hosts:
        belongs_to:
          - hosts

* Place related ``cinder-backup`` configuration to
  ``/etc/openstack_deploy/group_vars/cinder_backup.yml``:

  .. code-block:: yaml

    ---
    cinder_service_backup_program_enabled: True
    cinder_service_backup_driver: cinder.backup.drivers.nfs.NFSBackupDriver
    cinder_cinder_conf_overrides:
      DEFAULT:
        backup_share: "nfs01:/data/cinder-backup"
        backup_mount_options: rw,nolock,noatime

* Run playbooks to create containers and cinder-backup service

  .. code-block:: console

    openstack-ansible openstack.osa.containers_lxc_create --limit cinder_backup,lxc_hosts
    openstack-ansible openstack.osa.cinder --limit cinder_backup
