pgpool2
=========

Install and configure pgpool2

Requirements
------------

None

Role Variables
--------------

postgresql_version: 10
postgresql_archive_directory: "/var/lib/pgsql/archivedir"
postgresql_data_directory: /var/lib/pgsql/10/data
postgresql_install_repository: true
pgpool2_templates_version: 4.1 - Nedded for template path.
pgpool2_do_online_recovery: False - If you want to automatically recover standby nodes set True
pgpool2_pcp_user_name: pcpAdmin - PCP account
pgpool2_pcp_user_password: Password01

pgpool2_backends: [] - Use short hostname not ip.

Other variables in `defaults/main.yml`

Dependencies
------------

2 or 3 postgres nodes.
I use role anxs.postgresql:

```
---
- hosts: all
  become: yes
  vars:
    postgresql_version: 10
    postgresql_listen_addresses:
      - '*'
    postgresql_archive_mode: 'on'
    postgresql_archive_command: 'cp "%p" "/var/lib/pgsql/archivedir/%f"'
    postgresql_archive_directory: "/var/lib/pgsql/archivedir"
    postgresql_max_wal_senders: 10
    postgresql_max_replication_slots: 10
    postgresql_wal_level: 'replica'
    postgresql_hot_standby: 'on'
    postgresql_wal_log_hints: 'on'
    postgresql_conf_directory: "{{ postgresql_data_directory }}"
    postgresql_users:
      - name: pgpool
        pass: md5039da4166f728631cf4c0f77a94c4ed5
        encrypted: yes
        groups:
          - pg_monitor
      - name: repl
        pass: md509dd76719cafeb56a0d32fe1cf5f04c1
        encrypted: yes
      - name: postgres
        pass: md5d17ea122f200c78f655c5d8dcf49eac1
        encrypted: yes
    postgresql_user_privileges:
      - name: pgpool
        role_attr_flags: LOGIN
      - name: repl
        role_attr_flags: REPLICATION,LOGIN

  roles:
    - eusebium.anxs.postgresql

  post_tasks:
    - name: Add all to pg_hba.conf
      lineinfile:
        path: /var/lib/pgsql/10/data/pg_hba.conf
        line: 'host    all             all             samenet                 md5'

    - name: Add replication to pg_hba.conf
      lineinfile:
        path: /var/lib/pgsql/10/data/pg_hba.conf
        line: 'host    replication     all             samenet                 md5'

    - name: Create .pgpass replication
      lineinfile:
        dest: ~/.pgpass
        line: '{{ ansible_hostname }}:5432:replication:repl:Password01'
        create: yes
        mode: 0600
      become_user: postgres
      delegate_to: "{{ item }}"
      loop: "{{ ansible_play_batch }}"

    - name: Create .pgpass postgres
      lineinfile:
        dest: ~/.pgpass
        line: '{{ ansible_hostname }}:5432:postgres:postgres:Password01'
        create: yes
        mode: 0600
      become_user: postgres
      delegate_to: "{{ item }}"
      loop: "{{ ansible_play_batch }}"

    - name: Update /etc/hosts
      lineinfile:
        path: /etc/hosts
        line: "{{ item }}"
      loop:
        - '192.168.10.11 p1'
        - '192.168.10.12 p2'
        - '192.168.10.13 p3'

    - name: Restart postgres
      service:
        name: postgresql-10
        state: restarted

```

SSH passwordless:

```
---
- hosts: all
  roles:
    - role: eusebium.ssh_keys
      ssh_keys_keypair_name: id_rsa_pgpool
      ssh_keys_keypair_owner: postgres
      ssh_keys_authorized_keys_owner: postgres

- hosts: all
  roles:
    - role: eusebium.ssh_keys
      ssh_keys_keypair_name: id_rsa
      ssh_keys_keypair_owner: postgres
      ssh_keys_authorized_keys_owner: postgres

- hosts: all
  roles:
    - role: eusebium.ssh_keys
      ssh_keys_keypair_name: id_rsa_pgpool
      ssh_keys_keypair_owner: root
      ssh_keys_authorized_keys_owner: postgres
```


Example Playbook
----------------



```
---
- hosts: all
  become: yes
  vars:
    pgpool2_do_online_recovery: True
    pgpool2_listen_addresses: '*'
    pgpool2_sr_check_user: pgpool
    pgpool2_pid_file_name: /var/run/postgresql/pgpool.pid
    pgpool2_health_check_period: 5
    pgpool2_health_check_timeout: 30
    pgpool2_health_check_user: pgpool
    pgpool2_health_check_max_retries: 3
    pgpool2_failover_command: '/etc/pgpool-II-10/failover.sh %d %h %p %D %m %H %M %P %r %R %N %S'
    pgpool2_follow_master_command: '/etc/pgpool-II-10/follow_master.sh %d %h %p %D %m %M %H %P %r %R'
    pgpool2_recovery_user: postgres
    pgpool2_recovery_password: 'Password01'
    pgpool2_recovery_1st_stage_command: 'recovery_1st_stage'
    pgpool2_enable_pool_hba: on
    pgpool2_use_watchdog: on
    pgpool2_delegate_IP: '192.168.10.10'
    pgpool2_if_up_cmd: '/usr/bin/sudo /sbin/ip addr add $_IP_$/24 dev eth1 label eth1:0'
    pgpool2_if_down_cmd: '/usr/bin/sudo /sbin/ip addr del $_IP_$/24 dev eth1'
    pgpool2_arping_cmd: '/usr/bin/sudo /usr/sbin/arping -U $_IP_$ -w 1 -I eth1'
    pgpool2_wd_hostname: '{{ ansible_eth1.ipv4.address }}'
    pgpool2_wd_port: 9000
    pgpool2_log_destination: 'syslog,stderr'
    pgpool2_syslog_facility: 'LOCAL1'
    pgpool2_backends:
      - hostname: p1
        port: 5432
        weight: 1
        data_directory: /var/lib/pgsql/10/data
        flag: ALLOW_TO_FAILOVER
        application_name: p1
      - hostname: p2
        port: 5432
        weight: 1
        data_directory: /var/lib/pgsql/10/data
        flag: ALLOW_TO_FAILOVER
        application_name: p2
      - hostname: p3
        port: 5432
        weight: 1
        data_directory: /var/lib/pgsql/10/data
        flag: ALLOW_TO_FAILOVER
        application_name: p3
    pgpool2_heartbeat_nodes:
      - hostname: p1
        address: 192.168.10.11
        port: 9694
        device: ''
      - hostname: p2
        address: 192.168.10.12
        port: 9694
        device: ''
      - hostname: p3
        address: 192.168.10.13
        port: 9694
        device: ''
    pgpool2_query_nodes:
      - hostname: p1
        address: 192.168.10.11
        port: 9999
        wd_port: 9000
      - hostname: p2
        address: 192.168.10.12
        port: 9999
        wd_port: 9000
      - hostname: p3
        address: 192.168.10.13
        port: 9999
        wd_port: 9000
    pgpool2_hba_custom:
      - comment: Connect pgpool
        type: host
        database: all
        user: pgpool
        address: "0.0.0.0/0"
        method: md5
      - comment: Connect postgres
        type: host
        database: all
        user: postgres
        address: "0.0.0.0/0"
        method: md5
    pgpool2_pool_users:
      - name: postgres
        pass: Password01
      - name: pgpool
        pass: Password01
    pgpool2_pcp_user_name: pcpAdmin
    pgpool2_pcp_user_password: Password01
    # pgpool2_num_init_children: 1
    # pgpool2_log_error_verbosity: verbose
    # pgpool2_client_min_messages: debug5
    # pgpool2_log_min_messages: debug5
    # pgpool2_log_connections: on
    # pgpool2_log_hostname: on
    # pgpool2_log_statement: on
    # pgpool2_log_per_node_statement: on
    # pgpool2_log_client_messages: on

  roles:
    - role: eusebium.pgpool2

```

License
-------

MIT

Author Information
------------------

E.Cristea
