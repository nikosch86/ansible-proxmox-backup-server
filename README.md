Ansible Role: Proxmox Backup Server
=========

Installs containerized proxmox backup server. using the proxmox backup server docker image (https://github.com/nikosch86/docker-proxmox-backup-server).  
Three users are created, an admin user for general management, a backup user to be used by consumers of the backup server and a monitoring user to be used by the monitoring tool.  
A token for monitoring is created, upon first execution of this role, the token is displayed as debug output.  

This role goes together well with the zabbix template (https://github.com/nikosch86/zabbix-proxmox-backup-server).

Requirements
------------

None.

Role Variables
--------------

Available variables are listed below, along with default values (see `defaults/main.yml`):

`pbs_service_dir: /opt/pbs`  
The directory of the docker compose deployment for this service.  

`pbs_service_user: root`  
The user that runs the docker compose deployment for this service.

`pbs_storage_dir: /data/pbs`  
The directory where the backups are stored.

`pbs_datastore: backups`  
The name of the datastore, this is only cosmetic.

`pbs_ssl_certificate: false`  
The path to the SSL certificate.  
It defaults to false, which will cause pbs to create a self signed certificate.

`pbs_ssl_certificate_key: false`  
The path to the SSL certificate key.  
It defaults to false, which will cause pbs to create a self signed certificate.

`pbs_timezone: "Asia/Dubai"`  
The timezone of the server.

`pbs_user_admin_password: "change_me"`  
The password for the admin user.

`pbs_user_backup_password: "change_me"`  
The password for the backup user.

`pbs_user_monitoring_password: "change_me"`  
The password for the monitoring user.

`pbs_network_mode: "port"`  
This variable can be used to control the network mode of the container.  
Setting it to "host" will allow you to control access using the host firewall.  

`pbs_disk_identity: false`  
Opt-in flag that enables PBS disk management (the `Administration → Disks` UI, the
`disks/list` API and `proxmox-backup-manager disk list`) with full SMART and udev
identity (model/serial/wwn). When `true` it bind-mounts `/run/udev:/run/udev:ro` (for
udev identity) **and** `/dev:/dev`, and adds the `SYS_RAWIO` and `SYS_ADMIN`
capabilities. The two bind mounts and the capabilities are unioned (and de-duplicated)
with `pbs_extra_volumes` and `pbs_cap_add`, so the flag is the single source of truth and
those escape hatches stay available for anything extra.  
The `/dev:/dev` bind is required on PBS 4.2+: with only `/run/udev` bound, PBS enumerates
every device in the host's shared `/sys/block` and hard-`statx`es `/dev/<name>` for each,
which `ENOENT`s for any device that is not passed through (loop\*/dm-\*/md\*/non-passed
disks) and aborts the whole list with HTTP 400. Binding `/dev` supplies the inode so the
`statx` succeeds; actual I/O stays least-privilege because the device cgroup
(`pbs_devices`) still gates every read/write — the bind only provides inodes.  
Pair this with `pbs_devices` to grant SMART access to the specific disks you care about.  

`pbs_devices: []`  
A list of host devices to expose to the container, in docker compose `devices:` syntax.  
This is the per-host allow-list that gates real SMART I/O, e.g. `["/dev/sda", "/dev/nvme0n1"]`.  
Typically used together with `pbs_disk_identity: true`.  

`pbs_cap_add: []`  
A list of Linux capabilities to add to the container, in docker compose `cap_add:` syntax.  
For SMART access on raw disks this is `["SYS_RAWIO", "SYS_ADMIN"]` — both are added
automatically by `pbs_disk_identity: true`, so this is only needed for additional capabilities.  

`pbs_extra_volumes: []`  
A list of additional volume mounts appended to the container, in docker compose `volumes:` syntax.  
The udev runtime and `/dev` needed for disk identity are added automatically by
`pbs_disk_identity: true`; use this only for additional mounts.  

Dependencies
------------
Docker needs to be installed, use the `geerlingguy.docker` for example.


Example Playbook
----------------

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```
    - hosts: servers
      become: true
      
      roles:
      - role: pbs
        tags: [deployment]
        vars:
          pbs_service_dir: "/home/{{ service_user }}/container/pbs"
          pbs_storage_dir: /mnt/data/pbs
          pbs_ssl_certificate: "/etc/ssl/private/server.crt"
          pbs_ssl_certificate_key: "/etc/ssl/private/server.key"
          pbs_user_admin_password: "{{ admin_password_from_vault }}"
```

Please note that a monitoring user, along with a token is generated, it will only be shown upon first execution of the role.  

License
-------

MIT

Author Information
------------------

https://github.com/nikosch86
