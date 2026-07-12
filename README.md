Ansible Role: Proxmox Backup Server
=========

Installs containerized proxmox backup server. using the proxmox backup server docker image (https://github.com/nikosch86/docker-proxmox-backup-server).  
Three users are created, an admin user for general management, a backup user to be used by consumers of the backup server and a monitoring user to be used by the monitoring tool.  
A token for monitoring is created, upon first execution of this role, the token is displayed as debug output.  

This role goes together well with the zabbix template (https://github.com/nikosch86/zabbix-proxmox-backup-server).

Requirements
------------

None.

Subscription
------------

Nodes deployed by this role are **unsubscribed by design** — a containerized PBS cannot
hold a subscription. PBS reports `status: notfound` permanently, and monitoring will flag
it. This is expected, not a misconfiguration: the subscription buys Proxmox's paid support,
and PBS itself is fully functional without one.  
If you monitor with the companion zabbix template, it alerts on subscription state by
default; set `{$PBS.SUBSCRIPTION.STATE.ACTIVE}` to `notfound` to silence it.

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

`pbs_root_password: false`  
The password for the `root@pam` superuser. Defaults to `false`, which leaves the image
default untouched.  
`root@pam` is PAM-backed, so `proxmox-backup-manager` cannot set it — the role pipes
`chpasswd` into the container instead. The container's `/etc/shadow` is *not* on a
persisted volume, so it resets to the image default whenever the container is recreated;
the role therefore re-applies this on every converge, which makes it self-healing. Set it
from a vaulted variable.  

`pbs_user_admin_password: "change_me"`  
The password for the admin user.  
Note the `@pbs` user passwords are only applied **when the user is created**. Changing
this on an existing deployment does not rotate the password — do that in the PBS UI or
with `proxmox-backup-manager`.

`pbs_user_backup_password: "change_me"`  
The password for the backup user.

`pbs_user_monitoring_password: "change_me"`  
The password for the monitoring user.

`pbs_network_mode: "port"`  
This variable can be used to control the network mode of the container.  
Setting it to "host" will allow you to control access using the host firewall.  

`pbs_disk_identity: false`  
Opt-in flag that enables PBS disk *reporting* (the `Administration → Disks` UI, the
`disks/list` API and `proxmox-backup-manager disk list`) with full SMART and udev
identity (model/serial/wwn). When `true` it bind-mounts `/run/udev:/run/udev:ro` (for
udev identity) **and** `/dev:/dev`, adds the `SYS_RAWIO` and `SYS_ADMIN`
capabilities, and grants read-only access to the host's block devices. The two bind
mounts and the capabilities are unioned (and de-duplicated) with `pbs_extra_volumes` and
`pbs_cap_add`, so the flag is the single source of truth and those escape hatches stay
available for anything extra.  
The default is deliberately read-only: it covers SMART, wearout and `disk list`, but
**not** the disk-*provisioning* actions in `Administration → Disks` (initialize GPT,
create a datastore on a disk), which need write. If you provision disks from the PBS UI,
opt in with `pbs_device_cgroup_rules: ["b *:* rmw"]`.  
The `/dev:/dev` bind is required on PBS 4.2+: with only `/run/udev` bound, PBS enumerates
every device in the host's shared `/sys/block` and hard-`statx`es `/dev/<name>` for each,
which `ENOENT`s for any device that is not passed through (loop\*/dm-\*/md\*/non-passed
disks) and aborts the whole list with HTTP 400. Binding `/dev` supplies the inode so the
`statx` succeeds — but a bind mount only supplies the inode, never the device-cgroup
permission, so `open()` on a disk still fails with `Operation not permitted` until
something grants that permission.  
By default the flag therefore also emits `device_cgroup_rules: ['b *:* r']` — read-only
access to every block device — which is what makes SMART actually work out of the box.
The wildcard default applies only when both `pbs_devices` and `pbs_device_cgroup_rules`
are empty; setting either one replaces it.  

`pbs_devices: []`  
A list of host devices to expose to the container, in docker compose `devices:` syntax,
e.g. `["/dev/sda", "/dev/nvme0n1"]`. Docker grants each listed device `rwm` cgroup
permission. Setting it suppresses the read-only wildcard default.  
Note this is *more* privileged per disk than the default (`rwm` vs `r`) and it goes
stale: a disk added, removed or renamed across a reboot silently drops out of the list,
and SMART for it goes back to `unknown`. Prefer `pbs_device_cgroup_rules` unless you
specifically need to pin the container to an exact set of disks.  

`pbs_device_cgroup_rules: []`  
A list of device cgroup rules, in docker compose `device_cgroup_rules:` syntax. Setting
it replaces the default that `pbs_disk_identity: true` would otherwise apply.  
Use `["b *:* rmw"]` to add write access for disk provisioning from the PBS UI.  

`pbs_cap_add: []`  
A list of Linux capabilities to add to the container, in docker compose `cap_add:` syntax.  
For SMART access on raw disks this is `["SYS_RAWIO", "SYS_ADMIN"]` — both are added
automatically by `pbs_disk_identity: true`, so this is only needed for additional capabilities.  

`pbs_extra_volumes: []`  
A list of additional volume mounts appended to the container, in docker compose `volumes:` syntax.  
The udev runtime and `/dev` needed for disk identity are added automatically by
`pbs_disk_identity: true`; use this only for additional mounts.  

`pbs_acl_lines:`  
The ACL entries written to `etc/acl.cfg` (one `lineinfile` per entry). Defaults to the four
built-in roles:  
```yaml
pbs_acl_lines:
  - "acl:1:/:admin@pbs:Admin"
  - "acl:1:/:backup@pbs,sync@pbs:DatastorePowerUser"
  - "acl:1:/:monitoring@pbs,monitoring@pbs!zabbix:Audit"
  - "acl:1:/:sync@pbs:DatastoreReader"
```
Override it to change the managed ACL lines, or set it to `[]` to make the task a no-op and
cede ownership of `acl.cfg` entirely (e.g. if you template-manage the file yourself).  

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
