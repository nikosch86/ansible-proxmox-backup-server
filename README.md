Ansible Role: Proxmox Backup Server
=========

Installs containerized proxmox backup server. using the proxmox backup server docker image (https://github.com/nikosch86/docker-proxmox-backup-server).  
Three users are created, an admin user for general management, a backup user to be used by consumers of the backup server and a monitoring user to be used by the monitoring tool.  
A token for monitoring is created, upon first execution of this role, the token is displayed as debug output.  

This role goes together well with the zabbix template (https://github.com/nikosch86/zabbix-proxmox-backup-server).

Requirements
------------

Any pre-requisites that may not be covered by Ansible itself or the role should be mentioned here. For instance, if the role uses the EC2 module, it may be a good idea to mention in this section that the boto package is required.

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

Dependencies
------------
None.


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
