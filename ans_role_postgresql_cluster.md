Ниже представлена полная ansible роль для развёртывания кластера PostgreSQL 17 с Patroni и etcd (2 ноды БД + свидетель). Роль содержит все необходимые файлы, задачи, шаблоны и переменные. Она рассчитана на Ubuntu 22.04/24.04 или Debian 12.

Структура каталога роли

```
postgres-patroni-cluster/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── install_postgres.yml
│   ├── install_etcd.yml
│   ├── install_patroni.yml
│   ├── configure_etcd.yml
│   ├── configure_patroni.yml
│   └── start_services.yml
├── templates/
│   ├── etcd.conf.j2
│   ├── etcd.service.j2
│   ├── patroni.yml.j2
│   └── patroni.service.j2
└── vars/
    └── main.yml
```

---

1. defaults/main.yml

```yaml
---
# Версии компонентов
postgresql_version: "17"
etcd_version: "3.5.16"
patroni_version: "4.0.4"

# Параметры кластера
cluster_name: "pg-cluster"
etcd_initial_cluster_token: "etcd-cluster-token"
etcd_data_dir: "/var/lib/etcd"
postgresql_data_dir: "/var/lib/postgresql/{{ postgresql_version }}/main"

# Порты
etcd_client_port: 2379
etcd_peer_port: 2380
postgresql_port: 5432
patroni_restapi_port: 8008

# Настройки etcd
etcd_heartbeat_interval: 1000
etcd_election_timeout: 5000
etcd_initial_cluster_state: "new"   # после первого запуска можно изменить на "existing"

# Учётные данные PostgreSQL / Patroni
patroni_superuser: "postgres"
patroni_superuser_password: "strongpassword"
patroni_replication_user: "replicator"
patroni_replication_password: "replpassword"
patroni_rewind_user: "rewinduser"
patroni_rewind_password: "rewindpassword"

# REST API Patroni
patroni_restapi_listen: "0.0.0.0:{{ patroni_restapi_port }}"

# Списки хостов (переопределяются в inventory)
etcd_members: []           # [{ name: "node1", ip: "192.168.1.11" }, ...]
postgres_nodes: []         # ["node1", "node2"]

# Пути
etcd_binary_install_dir: "/usr/local/bin"
patroni_venv_dir: "/opt/patroni-venv"
patroni_config_dir: "/etc/patroni"
etcd_config_dir: "/etc/etcd"
systemd_service_dir: "/etc/systemd/system"
```

---

2. vars/main.yml

```yaml
---
# Внутренние переменные, которые не должны переопределяться пользователем
etcd_client_url: "http://{{ ansible_default_ipv4.address }}:{{ etcd_client_port }}"
etcd_peer_url: "http://{{ ansible_default_ipv4.address }}:{{ etcd_peer_port }}"
patroni_connect_address: "{{ ansible_default_ipv4.address }}:{{ patroni_restapi_port }}"
```

---

3. meta/main.yml

```yaml
---
galaxy_info:
  author: "Your Name"
  description: "Deploy PostgreSQL 17 HA cluster with Patroni and etcd (2 DB nodes + witness)"
  license: "MIT"
  min_ansible_version: "2.12"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
    - name: Debian
      versions:
        - bookworm
  galaxy_tags:
    - postgresql
    - patroni
    - etcd
    - highavailability
    - cluster
dependencies: []
```

---

4. tasks/main.yml

```yaml
---
- name: Include OS-specific variables (if any)
  include_vars: "{{ ansible_os_family }}.yml"
  ignore_errors: yes

- name: Install etcd on all nodes
  include_tasks: install_etcd.yml
  when: inventory_hostname in groups['etcd_cluster']

- name: Install PostgreSQL on database nodes
  include_tasks: install_postgres.yml
  when: inventory_hostname in groups['postgres_cluster']

- name: Install Patroni on database nodes
  include_tasks: install_patroni.yml
  when: inventory_hostname in groups['postgres_cluster']

- name: Configure etcd on all nodes
  include_tasks: configure_etcd.yml
  when: inventory_hostname in groups['etcd_cluster']

- name: Configure Patroni on database nodes
  include_tasks: configure_patroni.yml
  when: inventory_hostname in groups['postgres_cluster']

- name: Start services
  include_tasks: start_services.yml
```

---

5. tasks/install_etcd.yml

```yaml
---
- name: Create etcd system user
  ansible.builtin.user:
    name: etcd
    system: yes
    shell: /usr/sbin/nologin
    home: "{{ etcd_data_dir }}"
    create_home: no

- name: Create etcd data directory
  ansible.builtin.file:
    path: "{{ etcd_data_dir }}"
    state: directory
    owner: etcd
    group: etcd
    mode: '0750'

- name: Download etcd binary
  ansible.builtin.get_url:
    url: "https://github.com/etcd-io/etcd/releases/download/v{{ etcd_version }}/etcd-v{{ etcd_version }}-linux-amd64.tar.gz"
    dest: "/tmp/etcd-v{{ etcd_version }}-linux-amd64.tar.gz"
    mode: '0644'

- name: Extract etcd archive
  ansible.builtin.unarchive:
    src: "/tmp/etcd-v{{ etcd_version }}-linux-amd64.tar.gz"
    dest: /tmp
    remote_src: yes
    creates: "/tmp/etcd-v{{ etcd_version }}-linux-amd64"

- name: Install etcd binaries
  ansible.builtin.copy:
    src: "/tmp/etcd-v{{ etcd_version }}-linux-amd64/{{ item }}"
    dest: "{{ etcd_binary_install_dir }}/{{ item }}"
    owner: root
    group: root
    mode: '0755'
    remote_src: yes
  loop:
    - etcd
    - etcdctl
    - etcdutl

- name: Clean up downloaded files
  ansible.builtin.file:
    path: "{{ item }}"
    state: absent
  loop:
    - "/tmp/etcd-v{{ etcd_version }}-linux-amd64.tar.gz"
    - "/tmp/etcd-v{{ etcd_version }}-linux-amd64"

- name: Create etcd configuration directory
  ansible.builtin.file:
    path: "{{ etcd_config_dir }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
```

---

6. tasks/install_postgres.yml

```yaml
---
- name: Install required packages for PGDG repository
  ansible.builtin.apt:
    name:
      - gnupg
      - wget
      - lsb-release
      - ca-certificates
    state: present
    update_cache: yes

- name: Add PostgreSQL official repository key
  ansible.builtin.apt_key:
    url: "https://www.postgresql.org/media/keys/ACCC4CF8.asc"
    state: present

- name: Add PostgreSQL repository
  ansible.builtin.apt_repository:
    repo: "deb http://apt.postgresql.org/pub/repos/apt {{ ansible_distribution_release }}-pgdg main"
    state: present
    update_cache: yes

- name: Install PostgreSQL {{ postgresql_version }}
  ansible.builtin.apt:
    name:
      - "postgresql-{{ postgresql_version }}"
      - "postgresql-contrib-{{ postgresql_version }}"
      - "postgresql-client-{{ postgresql_version }}"
    state: present

- name: Stop and disable default PostgreSQL service (Patroni will manage it)
  ansible.builtin.systemd:
    name: "postgresql@{{ postgresql_version }}-main"
    state: stopped
    enabled: no
  ignore_errors: yes

- name: Ensure PostgreSQL data directory exists and is owned by postgres
  ansible.builtin.file:
    path: "{{ postgresql_data_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: '0700'
```

---

7. tasks/install_patroni.yml

```yaml
---
- name: Install Python and virtualenv tools
  ansible.builtin.apt:
    name:
      - python3
      - python3-pip
      - python3-venv
      - python3-dev
      - libpq-dev
      - build-essential
    state: present

- name: Create virtual environment for Patroni
  ansible.builtin.command:
    cmd: "python3 -m venv {{ patroni_venv_dir }}"
    creates: "{{ patroni_venv_dir }}"

- name: Ensure virtualenv is owned by postgres user
  ansible.builtin.file:
    path: "{{ patroni_venv_dir }}"
    state: directory
    owner: postgres
    group: postgres
    recurse: yes

- name: Install Patroni and dependencies inside virtualenv
  ansible.builtin.pip:
    name:
      - "patroni[etcd]=={{ patroni_version }}"
      - "psycopg2-binary"
    virtualenv: "{{ patroni_venv_dir }}"
    virtualenv_python: python3
  become_user: postgres

- name: Create Patroni configuration directory
  ansible.builtin.file:
    path: "{{ patroni_config_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: '0755'
```

---

8. tasks/configure_etcd.yml

```yaml
---
- name: Generate etcd configuration file
  ansible.builtin.template:
    src: etcd.conf.j2
    dest: "{{ etcd_config_dir }}/etcd.conf"
    owner: root
    group: etcd
    mode: '0640'
  notify: restart etcd

- name: Generate etcd systemd unit file
  ansible.builtin.template:
    src: etcd.service.j2
    dest: "{{ systemd_service_dir }}/etcd.service"
    owner: root
    group: root
    mode: '0644'
  notify: restart etcd

- name: Reload systemd daemon
  ansible.builtin.systemd:
    daemon_reload: yes
```

---

9. tasks/configure_patroni.yml

```yaml
---
- name: Generate Patroni configuration file
  ansible.builtin.template:
    src: patroni.yml.j2
    dest: "{{ patroni_config_dir }}/patroni.yml"
    owner: postgres
    group: postgres
    mode: '0640'
  notify: restart patroni

- name: Generate Patroni systemd unit file
  ansible.builtin.template:
    src: patroni.service.j2
    dest: "{{ systemd_service_dir }}/patroni.service"
    owner: root
    group: root
    mode: '0644'
  notify: restart patroni

- name: Reload systemd daemon
  ansible.builtin.systemd:
    daemon_reload: yes
```

---

10. tasks/start_services.yml

```yaml
---
- name: Start etcd service on all etcd nodes
  ansible.builtin.systemd:
    name: etcd
    state: started
    enabled: yes
  when: inventory_hostname in groups['etcd_cluster']

- name: Wait for etcd cluster to be healthy
  ansible.builtin.command:
    cmd: "etcdctl --endpoints={% for m in etcd_members %}http://{{ m.ip }}:{{ etcd_client_port }}{% if not loop.last %},{% endif %}{% endfor %} endpoint health"
  register: etcd_health
  until: etcd_health.rc == 0
  retries: 30
  delay: 5
  changed_when: false
  when: inventory_hostname in groups['etcd_cluster'] and inventory_hostname == groups['etcd_cluster'][0]

- name: Start Patroni service on database nodes
  ansible.builtin.systemd:
    name: patroni
    state: started
    enabled: yes
  when: inventory_hostname in groups['postgres_cluster']

- name: Wait for Patroni to initialise PostgreSQL cluster
  ansible.builtin.command:
    cmd: "{{ patroni_venv_dir }}/bin/patronictl -c {{ patroni_config_dir }}/patroni.yml list"
  register: patroni_status
  until: patroni_status.rc == 0
  retries: 30
  delay: 5
  changed_when: false
  when: inventory_hostname in groups['postgres_cluster'] and inventory_hostname == groups['postgres_cluster'][0]
```

---

11. handlers/main.yml

```yaml
---
- name: restart etcd
  ansible.builtin.systemd:
    name: etcd
    state: restarted
    daemon_reload: yes

- name: restart patroni
  ansible.builtin.systemd:
    name: patroni
    state: restarted
    daemon_reload: yes
```

---

12. Шаблоны

templates/etcd.conf.j2

```ini
# {{ ansible_managed }}
name: {{ inventory_hostname }}
data-dir: {{ etcd_data_dir }}
listen-client-urls: http://{{ ansible_default_ipv4.address }}:{{ etcd_client_port }}
advertise-client-urls: http://{{ ansible_default_ipv4.address }}:{{ etcd_client_port }}
listen-peer-urls: http://{{ ansible_default_ipv4.address }}:{{ etcd_peer_port }}
initial-advertise-peer-urls: http://{{ ansible_default_ipv4.address }}:{{ etcd_peer_port }}
initial-cluster: {% for member in etcd_members %}{{ member.name }}=http://{{ member.ip }}:{{ etcd_peer_port }}{% if not loop.last %},{% endif %}{% endfor %}
initial-cluster-state: {{ etcd_initial_cluster_state }}
initial-cluster-token: {{ etcd_initial_cluster_token }}
heartbeat-interval: {{ etcd_heartbeat_interval }}
election-timeout: {{ etcd_election_timeout }}
```

templates/etcd.service.j2

```ini
[Unit]
Description=etcd key-value store
Documentation=https://etcd.io/docs
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
Group=etcd
EnvironmentFile=-{{ etcd_config_dir }}/etcd.conf
ExecStart={{ etcd_binary_install_dir }}/etcd
Restart=always
RestartSec=5s
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

templates/patroni.yml.j2

```yaml
scope: {{ cluster_name }}
namespace: /{{ cluster_name }}/
name: {{ inventory_hostname }}

restapi:
  listen: {{ patroni_restapi_listen }}
  connect_address: {{ patroni_connect_address }}

etcd:
  hosts: {% for member in etcd_members %}{{ member.ip }}:{{ etcd_client_port }}{% if not loop.last %},{% endif %}{% endfor %}

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 200
        max_worker_processes: 8
        max_parallel_workers: 8
        max_parallel_maintenance_workers: 4
        shared_buffers: 256MB
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        archive_mode: "on"
        archive_command: "/bin/true"
  initdb:
  - encoding: UTF8
  - data-checksums
  pg_hba:
  - host replication {{ patroni_replication_user }} 0.0.0.0/0 md5
  - host all {{ patroni_superuser }} 0.0.0.0/0 md5
  users:
    {{ patroni_superuser }}:
      password: {{ patroni_superuser_password }}
      options:
        - createrole
        - createdb
    {{ patroni_replication_user }}:
      password: {{ patroni_replication_password }}
      options:
        - replication
    {{ patroni_rewind_user }}:
      password: {{ patroni_rewind_password }}
      options:
        - replication

postgresql:
  listen: 0.0.0.0:{{ postgresql_port }}
  connect_address: {{ ansible_default_ipv4.address }}:{{ postgresql_port }}
  data_dir: {{ postgresql_data_dir }}
  bin_dir: /usr/lib/postgresql/{{ postgresql_version }}/bin
  pgpass: /tmp/pgpass
  authentication:
    replication:
      username: {{ patroni_replication_user }}
      password: {{ patroni_replication_password }}
    superuser:
      username: {{ patroni_superuser }}
      password: {{ patroni_superuser_password }}
    rewind:
      username: {{ patroni_rewind_user }}
      password: {{ patroni_rewind_password }}
  parameters:
    unix_socket_directories: '/var/run/postgresql'
    synchronous_mode: false
    synchronous_commit: "on"
    synchronous_standby_names: "*"
```

templates/patroni.service.j2

```ini
[Unit]
Description=Patroni PostgreSQL High Availability
After=network.target etcd.service
Wants=etcd.service

[Service]
Type=simple
User=postgres
Group=postgres
ExecStart={{ patroni_venv_dir }}/bin/patroni {{ patroni_config_dir }}/patroni.yml
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

---

13. Пример inventory

inventory/hosts.yml:

```yaml
all:
  children:
    etcd_cluster:
      hosts:
        node1:
          ansible_host: 192.168.1.11
        node2:
          ansible_host: 192.168.1.12
        witness:
          ansible_host: 192.168.1.13
    postgres_cluster:
      hosts:
        node1:
        node2:
```

inventory/group_vars/all.yml:

```yaml
etcd_members:
  - { name: node1, ip: 192.168.1.11 }
  - { name: node2, ip: 192.168.1.12 }
  - { name: witness, ip: 192.168.1.13 }

postgres_nodes:
  - node1
  - node2
```

---

14. Пример плейбука

playbook.yml:

```yaml
---
- name: Deploy PostgreSQL HA cluster with Patroni and etcd
  hosts: all
  become: yes
  roles:
    - postgres-patroni-cluster
```

---

15. Проверка после развёртывания

1. Проверка etcd (на любой etcd-ноде):
   ```bash
   etcdctl --endpoints=http://node1:2379,http://node2:2379,http://witness:2379 endpoint health
   ```
   Все три должны быть healthy.
2. Проверка Patroni (на ноде с БД):
   ```bash
   /opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml list
   ```
   Вывод должен показывать одну ноду как Leader, другую как Replica.
3. Тест failover:
   Остановите Patroni на лидере:
   ```bash
   systemctl stop patroni
   ```
   Через 10–15 секунд вторая нода станет лидером.

---

Примечания

· Роль предполагает, что все хосты видят друг друга по именам, указанным в etcd_members. При необходимости добавьте записи в /etc/hosts.
· Пароли в примере заданы в открытом виде. В production рекомендуется использовать ansible-vault или внешнее хранилище секретов.
· После первого запуска кластера etcd рекомендуется изменить etcd_initial_cluster_state на existing в переменных, чтобы избежать случайной переинициализации.
· Для witness-ноды не выполняется установка и настройка PostgreSQL/Patroni, так как она не входит в группу postgres_cluster.

Данная роль полностью автоматизирует развёртывание отказоустойчивого кластера PostgreSQL 17 с Patroni и etcd.