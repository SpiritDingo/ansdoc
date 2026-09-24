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

Вот обновлённая Ansible-роль с поддержкой Oracle Linux 9, Ubuntu 22.04 и Ubuntu 24.04. Основные изменения коснулись структуры переменных (по семействам ОС) и задач установки.

📁 Обновлённая структура роли

```
postgres-patroni-cluster/
├── defaults/
│   └── main.yml
├── vars/
│   ├── main.yml
│   ├── Debian.yml          # Ubuntu 22.04 / 24.04
│   └── RedHat.yml          # Oracle Linux 9
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
├── handlers/
│   └── main.yml
└── meta/
    └── main.yml
```

---

1. vars/main.yml — загрузчик ОС-переменных

```yaml
---
# Загружаем переменные, специфичные для семейства ОС
- name: Load OS-specific variables
  ansible.builtin.include_vars: "{{ ansible_os_family }}.yml"
```

Этот файл вызывается в tasks/main.yml первой задачей, чтобы все последующие задачи уже имели доступ к нужным путям и именам пакетов.

---

2. vars/Debian.yml (Ubuntu 22.04 / 24.04)

```yaml
---
# Пакеты для установки PostgreSQL
postgresql_packages:
  - "postgresql-{{ postgresql_version }}"
  - "postgresql-contrib-{{ postgresql_version }}"
  - "postgresql-client-{{ postgresql_version }}"

# Пакеты для сборки/установки Patroni
patroni_build_packages:
  - python3
  - python3-pip
  - python3-venv
  - python3-dev
  - libpq-dev
  - build-essential

# Путь к бинарникам PostgreSQL
postgresql_bin_dir: "/usr/lib/postgresql/{{ postgresql_version }}/bin"

# Имя systemd-сервиса PostgreSQL (для остановки/отключения)
postgresql_service_name: "postgresql@{{ postgresql_version }}-main"

# Пакетный менеджер
package_manager: apt

# Зависимости для etcd (бинарная установка — не требуются)
etcd_dependencies: []
```

---

3. vars/RedHat.yml (Oracle Linux 9)

```yaml
---
# Пакеты для установки PostgreSQL (PGDG RHEL-репозиторий)
postgresql_packages:
  - "postgresql{{ postgresql_version }}"
  - "postgresql{{ postgresql_version }}-server"
  - "postgresql{{ postgresql_version }}-contrib"

# Пакеты для сборки/установки Patroni
patroni_build_packages:
  - python3
  - python3-pip
  - python3-devel
  - libpq-devel
  - gcc
  - make

# Путь к бинарникам PostgreSQL
postgresql_bin_dir: "/usr/pgsql-{{ postgresql_version }}/bin"

# Имя systemd-сервиса PostgreSQL
postgresql_service_name: "postgresql-{{ postgresql_version }}"

# Пакетный менеджер
package_manager: dnf

# Зависимости для etcd (бинарная установка — не требуются)
etcd_dependencies: []
```

---

4. Обновлённый tasks/main.yml

```yaml
---
- name: Load OS-specific variables
  ansible.builtin.include_vars: "{{ ansible_os_family }}.yml"

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

5. Обновлённый tasks/install_postgres.yml (кросс-платформенный)

```yaml
---
# ===== Debian / Ubuntu =====
- name: Add PostgreSQL repository key (Debian family)
  ansible.builtin.apt_key:
    url: "https://www.postgresql.org/media/keys/ACCC4CF8.asc"
    state: present
  when: ansible_os_family == "Debian"

- name: Add PostgreSQL repository (Debian family)
  ansible.builtin.apt_repository:
    repo: "deb http://apt.postgresql.org/pub/repos/apt {{ ansible_distribution_release }}-pgdg main"
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Install PostgreSQL (Debian family)
  ansible.builtin.apt:
    name: "{{ postgresql_packages }}"
    state: present
  when: ansible_os_family == "Debian"

# ===== RedHat / Oracle Linux =====
- name: Install PGDG repository RPM (RedHat family)
  ansible.builtin.dnf:
    name: "https://download.postgresql.org/pub/repos/yum/reporpms/EL-{{ ansible_distribution_major_version }}-x86_64/pgdg-redhat-repo-latest.noarch.rpm"
    state: present
    disable_gpg_check: yes
  when: ansible_os_family == "RedHat"

- name: Disable built-in PostgreSQL module (Oracle Linux 9)
  ansible.builtin.command:
    cmd: "dnf -qy module disable postgresql"
  changed_when: false
  when: ansible_os_family == "RedHat"

- name: Install PostgreSQL (RedHat family)
  ansible.builtin.dnf:
    name: "{{ postgresql_packages }}"
    state: present
  when: ansible_os_family == "RedHat"

# ===== Общие задачи =====
- name: Stop and disable default PostgreSQL service (Patroni will manage it)
  ansible.builtin.systemd:
    name: "{{ postgresql_service_name }}"
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

6. Обновлённый tasks/install_patroni.yml

```yaml
---
- name: Install Python and build dependencies
  ansible.builtin.package:
    name: "{{ patroni_build_packages }}"
    state: present

- name: Ensure pip is up to date
  ansible.builtin.pip:
    name: pip
    state: latest
    executable: pip3

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

Важно: ansible.builtin.pip требует наличия pip на целевом хосте. Для Oracle Linux 9 может потребоваться пакет python3-pip (он уже включён в patroni_build_packages).

---

7. Обновлённый tasks/install_etcd.yml (бинарная установка — одинаково для всех ОС)

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

8. Обновлённый tasks/start_services.yml

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

9. Обновлённый шаблон templates/patroni.yml.j2

Единственное изменение — путь к бинарникам PostgreSQL теперь берётся из переменной postgresql_bin_dir:

```yaml
postgresql:
  listen: 0.0.0.0:{{ postgresql_port }}
  connect_address: {{ ansible_default_ipv4.address }}:{{ postgresql_port }}
  data_dir: {{ postgresql_data_dir }}
  bin_dir: {{ postgresql_bin_dir }}
  pgpass: /tmp/pgpass
  # ... остальное без изменений
```

---

10. Обновлённый templates/etcd.service.j2

Для Oracle Linux 9 (SELinux) может потребоваться добавить контекст. Рекомендуется добавить строку SELinuxContext или использовать restorecon после установки. В остальном unit-файл универсален:

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

---

11. Обновлённый meta/main.yml

```yaml
---
galaxy_info:
  author: "Your Name"
  description: "Deploy PostgreSQL 17 HA cluster with Patroni and etcd (2 DB nodes + witness) on Oracle Linux 9, Ubuntu 22.04/24.04"
  license: "MIT"
  min_ansible_version: "2.12"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
    - name: EL
      versions:
        - "9"
  galaxy_tags:
    - postgresql
    - patroni
    - etcd
    - highavailability
    - cluster
dependencies: []
```

---

12. Пример inventory

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

group_vars/all.yml:

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

13. Ключевые различия для Oracle Linux 9

Параметр Ubuntu 22.04 / 24.04 Oracle Linux 9
Пакетный менеджер apt dnf
Репозиторий PGDG apt.postgresql.org pgdg-redhat-repo-latest.noarch.rpm (EL-9)
Пакеты PostgreSQL postgresql-17, postgresql-contrib-17 postgresql17, postgresql17-server, postgresql17-contrib
Путь к бинарникам /usr/lib/postgresql/17/bin /usr/pgsql-17/bin
systemd-сервис PG postgresql@17-main postgresql-17
Отключение встроенного модуля Не требуется dnf -qy module disable postgresql

Примечание: на Oracle Linux 9 SELinux может блокировать работу etcd/Patroni. Роль не отключает SELinux. При необходимости добавьте задачу с ansible.posix.selinux: state=permissive или настройте политики.

---

14. Запуск плейбука

```bash
ansible-playbook -i inventory/hosts.yml playbook.yml
```

Проверка выполняется так же, как описано ранее: etcdctl endpoint health и patronictl list.

Теперь роль автоматически определяет семейство ОС (ansible_os_family) и применяет соответствующие переменные и задачи, что позволяет использовать её на Oracle Linux 9 и Ubuntu 22.04/24.04 без изменений.

______________
______________


Полная Ansible роль: PostgreSQL 17 + Patroni + etcd (2 ноды + свидетель) с поддержкой Oracle Linux 9, Ubuntu 22.04 и Ubuntu 24.04

📁 Структура роли

```
postgres-patroni-cluster/
├── defaults/
│   └── main.yml
├── vars/
│   ├── Debian.yml
│   └── RedHat.yml
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
├── handlers/
│   └── main.yml
└── meta/
    └── main.yml
```

---

1. defaults/main.yml

```yaml
---
# ============================
# Версии компонентов
# ============================
postgresql_version: "17"
etcd_version: "3.5.16"
patroni_version: "4.0.4"

# ============================
# Параметры кластера
# ============================
cluster_name: "pg-cluster"
etcd_initial_cluster_token: "etcd-cluster-token"
etcd_data_dir: "/var/lib/etcd"
postgresql_data_dir: "/var/lib/pgsql/{{ postgresql_version }}/data"

# ============================
# Порты
# ============================
etcd_client_port: 2379
etcd_peer_port: 2380
postgresql_port: 5432
patroni_restapi_port: 8008

# ============================
# Настройки etcd
# ============================
etcd_heartbeat_interval: 1000
etcd_election_timeout: 5000
etcd_initial_cluster_state: "new"

# ============================
# Учётные данные PostgreSQL / Patroni
# ============================
patroni_superuser: "postgres"
patroni_superuser_password: "strongpassword"
patroni_replication_user: "replicator"
patroni_replication_password: "replpassword"
patroni_rewind_user: "rewinduser"
patroni_rewind_password: "rewindpassword"

# ============================
# REST API Patroni
# ============================
patroni_restapi_listen: "0.0.0.0:{{ patroni_restapi_port }}"

# ============================
# Списки хостов (переопределяются в inventory)
# ============================
etcd_members: []           # [{ name: "node1", ip: "192.168.1.11" }, ...]
postgres_nodes: []         # ["node1", "node2"]

# ============================
# Пути (кроссплатформенные)
# ============================
etcd_binary_install_dir: "/usr/local/bin"
patroni_venv_dir: "/opt/patroni-venv"
patroni_config_dir: "/etc/patroni"
etcd_config_dir: "/etc/etcd"
systemd_service_dir: "/etc/systemd/system"
postgresql_data_dir_debian: "/var/lib/postgresql/{{ postgresql_version }}/main"
postgresql_data_dir_redhat: "/var/lib/pgsql/{{ postgresql_version }}/data"
```

---

2. vars/Debian.yml (Ubuntu 22.04 / 24.04)

```yaml
---
# Пакеты PostgreSQL из PGDG
postgresql_packages:
  - "postgresql-{{ postgresql_version }}"
  - "postgresql-contrib-{{ postgresql_version }}"
  - "postgresql-client-{{ postgresql_version }}"

# Пакеты для сборки Patroni
patroni_build_packages:
  - python3
  - python3-pip
  - python3-venv
  - python3-dev
  - libpq-dev
  - build-essential
  - gnupg
  - wget
  - lsb-release
  - ca-certificates

# Путь к бинарникам PostgreSQL
postgresql_bin_dir: "/usr/lib/postgresql/{{ postgresql_version }}/bin"

# Имя systemd-сервиса PostgreSQL
postgresql_service_name: "postgresql@{{ postgresql_version }}-main"

# Директория данных PostgreSQL
postgresql_data_dir: "{{ postgresql_data_dir_debian }}"

# Пакетный менеджер
package_manager: apt

# Пакет для SELinux (не используется на Debian)
selinux_package: []
```

---

3. vars/RedHat.yml (Oracle Linux 9)

```yaml
---
# Пакеты PostgreSQL из PGDG (RHEL)
postgresql_packages:
  - "postgresql{{ postgresql_version }}"
  - "postgresql{{ postgresql_version }}-server"
  - "postgresql{{ postgresql_version }}-contrib"

# Пакеты для сборки Patroni
patroni_build_packages:
  - python3
  - python3-pip
  - python3-devel
  - libpq-devel
  - gcc
  - make
  - python3-virtualenv

# Путь к бинарникам PostgreSQL
postgresql_bin_dir: "/usr/pgsql-{{ postgresql_version }}/bin"

# Имя systemd-сервиса PostgreSQL
postgresql_service_name: "postgresql-{{ postgresql_version }}"

# Директория данных PostgreSQL
postgresql_data_dir: "{{ postgresql_data_dir_redhat }}"

# Пакетный менеджер
package_manager: dnf

# SELinux-утилиты
selinux_package:
  - policycoreutils-python-utils
  - libselinux-python3
```

---

4. meta/main.yml

```yaml
---
galaxy_info:
  author: "Your Name"
  description: >
    Deploy PostgreSQL 17 HA cluster with Patroni and etcd
    (2 DB nodes + witness) on Oracle Linux 9, Ubuntu 22.04/24.04.
  license: "MIT"
  min_ansible_version: "2.12"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
    - name: EL
      versions:
        - "9"
  galaxy_tags:
    - postgresql
    - patroni
    - etcd
    - highavailability
    - cluster
dependencies: []
```

---

5. handlers/main.yml

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

- name: reload systemd
  ansible.builtin.systemd:
    daemon_reload: yes
```

---

6. tasks/main.yml

```yaml
---
- name: Load OS-specific variables
  ansible.builtin.include_vars: "{{ ansible_os_family }}.yml"

- name: Fail if OS is not supported
  ansible.builtin.fail:
    msg: "Unsupported OS family: {{ ansible_os_family }}"
  when: ansible_os_family not in ['Debian', 'RedHat']

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

7. tasks/install_etcd.yml

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

8. tasks/install_postgres.yml

```yaml
---
# =========================================================
# Debian / Ubuntu
# =========================================================
- name: Install prerequisites for PGDG repository (Debian)
  ansible.builtin.apt:
    name:
      - gnupg
      - wget
      - lsb-release
      - ca-certificates
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Add PostgreSQL official repository key (Debian)
  ansible.builtin.apt_key:
    url: "https://www.postgresql.org/media/keys/ACCC4CF8.asc"
    state: present
  when: ansible_os_family == "Debian"

- name: Add PostgreSQL repository (Debian)
  ansible.builtin.apt_repository:
    repo: "deb http://apt.postgresql.org/pub/repos/apt {{ ansible_distribution_release }}-pgdg main"
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Install PostgreSQL (Debian)
  ansible.builtin.apt:
    name: "{{ postgresql_packages }}"
    state: present
  when: ansible_os_family == "Debian"

# =========================================================
# RedHat / Oracle Linux 9
# =========================================================
- name: Install PGDG repository RPM (RedHat)
  ansible.builtin.dnf:
    name: "https://download.postgresql.org/pub/repos/yum/reporpms/EL-{{ ansible_distribution_major_version }}-x86_64/pgdg-redhat-repo-latest.noarch.rpm"
    state: present
    disable_gpg_check: yes
  when: ansible_os_family == "RedHat"

- name: Disable built-in PostgreSQL module (Oracle Linux 9)
  ansible.builtin.command:
    cmd: "dnf -qy module disable postgresql"
  changed_when: false
  when: ansible_os_family == "RedHat"

- name: Install PostgreSQL (RedHat)
  ansible.builtin.dnf:
    name: "{{ postgresql_packages }}"
    state: present
  when: ansible_os_family == "RedHat"

# =========================================================
# Общие задачи
# =========================================================
- name: Stop and disable default PostgreSQL service
  ansible.builtin.systemd:
    name: "{{ postgresql_service_name }}"
    state: stopped
    enabled: no
  ignore_errors: yes

- name: Ensure PostgreSQL data directory exists
  ansible.builtin.file:
    path: "{{ postgresql_data_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: '0700'
```

---

9. tasks/install_patroni.yml

```yaml
---
- name: Install Python and build dependencies
  ansible.builtin.package:
    name: "{{ patroni_build_packages }}"
    state: present

- name: Ensure pip is up to date
  ansible.builtin.pip:
    name: pip
    state: latest
    executable: pip3

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

- name: Create symlink to patronictl for convenience
  ansible.builtin.file:
    src: "{{ patroni_venv_dir }}/bin/patronictl"
    dest: "/usr/local/bin/patronictl"
    state: link
  ignore_errors: yes
```

---

10. tasks/configure_etcd.yml

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

11. tasks/configure_patroni.yml

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

12. tasks/start_services.yml

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
  when:
    - inventory_hostname in groups['etcd_cluster']
    - inventory_hostname == groups['etcd_cluster'][0]

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
  when:
    - inventory_hostname in groups['postgres_cluster']
    - inventory_hostname == groups['postgres_cluster'][0]

- name: Show cluster status
  ansible.builtin.debug:
    msg: "{{ patroni_status.stdout_lines }}"
  when:
    - inventory_hostname in groups['postgres_cluster']
    - inventory_hostname == groups['postgres_cluster'][0]
    - patroni_status.stdout_lines is defined
```

---

13. templates/etcd.conf.j2

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

---

14. templates/etcd.service.j2

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

---

15. templates/patroni.yml.j2

```yaml
scope: {{ cluster_name }}
namespace: /{{ cluster_name }}/
name: {{ inventory_hostname }}

restapi:
  listen: {{ patroni_restapi_listen }}
  connect_address: {{ ansible_default_ipv4.address }}:{{ patroni_restapi_port }}

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
  bin_dir: {{ postgresql_bin_dir }}
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

---

16. templates/patroni.service.j2

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

17. Пример inventory

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

18. Пример playbook

playbook.yml:

```yaml
---
- name: Deploy PostgreSQL 17 HA cluster with Patroni and etcd
  hosts: all
  become: yes
  gather_facts: yes
  roles:
    - postgres-patroni-cluster
```

---

19. Запуск

```bash
# Проверка синтаксиса
ansible-playbook -i inventory/hosts.yml playbook.yml --syntax-check

# Запуск
ansible-playbook -i inventory/hosts.yml playbook.yml

# Запуск только для etcd-кластера
ansible-playbook -i inventory/hosts.yml playbook.yml --limit etcd_cluster

# Запуск только для PostgreSQL-нод
ansible-playbook -i inventory/hosts.yml playbook.yml --limit postgres_cluster
```

---

20. Проверка после развёртывания

Проверка etcd:

```bash
etcdctl --endpoints=http://node1:2379,http://node2:2379,http://witness:2379 endpoint health
etcdctl --endpoints=http://node1:2379,http://node2:2379,http://witness:2379 member list
```

Проверка Patroni:

```bash
/opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml list
```

Ожидаемый вывод:

```
+ Cluster: pg-cluster (7xxxxxxxxxxxxx) ----+----+-----------+
| Member | Host          | Role    | State   | TL | Lag in MB |
+--------+---------------+---------+---------+----+-----------+
| node1  | 192.168.1.11  | Leader  | running |  1 |           |
| node2  | 192.168.1.12  | Replica | running |  1 |         0 |
+--------+---------------+---------+---------+----+-----------+
```

Тест failover:

```bash
# Останавливаем Patroni на лидере
ansible node1 -b -m systemd -a "name=patroni state=stopped"

# Проверяем статус кластера (node2 должен стать лидером)
/opt/patroni-venv/bin/patronictl -c /etc/patroni/patroni.yml list

# Возвращаем ноду обратно
ansible node1 -b -m systemd -a "name=patroni state=started"
```

---

21. Особенности кроссплатформенной работы

Параметр Ubuntu 22.04/24.04 Oracle Linux 9
Пакетный менеджер apt dnf
Репозиторий PGDG apt.postgresql.org RPM pgdg-redhat-repo-latest.noarch.rpm
Пакеты PostgreSQL postgresql-17, postgresql-contrib-17 postgresql17-server, postgresql17-contrib
Путь к бинарникам /usr/lib/postgresql/17/bin /usr/pgsql-17/bin
Каталог данных /var/lib/postgresql/17/main /var/lib/pgsql/17/data
systemd-сервис postgresql@17-main postgresql-17
Отключение модуля не требуется dnf module disable postgresql
SELinux отсутствует может блокировать etcd/Patroni

Совет: если на Oracle Linux 9 SELinux блокирует работу etcd или Patroni, можно добавить задачу с ansible.posix.selinux: state=permissive или настроить политики SELinux.

---

22. Готовый состав роли

Все файлы выше образуют полноценную роль. Скопируйте их в структуру каталогов:

```
postgres-patroni-cluster/
├── defaults/main.yml
├── vars/Debian.yml
├── vars/RedHat.yml
├── meta/main.yml
├── handlers/main.yml
├── tasks/main.yml
├── tasks/install_etcd.yml
├── tasks/install_postgres.yml
├── tasks/install_patroni.yml
├── tasks/configure_etcd.yml
├── tasks/configure_patroni.yml
├── tasks/start_services.yml
├── templates/etcd.conf.j2
├── templates/etcd.service.j2
├── templates/patroni.yml.j2
└── templates/patroni.service.j2
```

Роль полностью готова к использованию и автоматически подстраивается под семейство ОС (ansible_os_family), что позволяет разворачивать кластер на Oracle Linux 9, Ubuntu 22.04 и Ubuntu 24.04 без изменения кода.

________
________
________


Обновлённая роль: работа через локальный Nexus

Ниже приведены только изменённые файлы. Остальные файлы (tasks/main.yml, handlers/main.yml, meta/main.yml, шаблоны etcd.conf.j2, etcd.service.j2, patroni.yml.j2, patroni.service.j2, tasks/configure_etcd.yml, tasks/configure_patroni.yml, tasks/start_services.yml) остаются без изменений.

Предполагается, что в Nexus уже настроены:

· Raw repository для хранения бинарника etcd — raw-etcd
· PyPI proxy — pypi-proxy
· APT proxy для apt.postgresql.org — apt-pgdg-proxy
· YUM proxy для download.postgresql.org — yum-pgdg-proxy

---

1. defaults/main.yml — добавлены переменные Nexus

```yaml
---
# ============================
# Версии компонентов
# ============================
postgresql_version: "17"
etcd_version: "3.5.16"
patroni_version: "4.0.4"

# ============================
# Nexus (уже настроен администратором)
# ============================
nexus_url: "https://nexus.example.com"
nexus_username: "ansible-reader"
nexus_password: "ChangeMe123!"

# Имена репозиториев в Nexus
nexus_repo_etcd_raw: "raw-etcd"
nexus_repo_pypi: "pypi-proxy"
nexus_repo_apt_pgdg: "apt-pgdg-proxy"
nexus_repo_yum_pgdg: "yum-pgdg-proxy"

# Производные URL
nexus_etcd_download_url: "{{ nexus_url }}/repository/{{ nexus_repo_etcd_raw }}/v{{ etcd_version }}/etcd-v{{ etcd_version }}-linux-amd64.tar.gz"
nexus_pypi_index_url: "{{ nexus_url }}/repository/{{ nexus_repo_pypi }}/simple"
nexus_pypi_trusted_host: "{{ nexus_url | urlsplit('hostname') }}"
nexus_apt_pgdg_url: "{{ nexus_url }}/repository/{{ nexus_repo_apt_pgdg }}"
nexus_yum_pgdg_url: "{{ nexus_url }}/repository/{{ nexus_repo_yum_pgdg }}"

# ============================
# Параметры кластера
# ============================
cluster_name: "pg-cluster"
etcd_initial_cluster_token: "etcd-cluster-token"
etcd_data_dir: "/var/lib/etcd"

# ============================
# Порты
# ============================
etcd_client_port: 2379
etcd_peer_port: 2380
postgresql_port: 5432
patroni_restapi_port: 8008

# ============================
# Настройки etcd
# ============================
etcd_heartbeat_interval: 1000
etcd_election_timeout: 5000
etcd_initial_cluster_state: "new"

# ============================
# Учётные данные PostgreSQL / Patroni
# ============================
patroni_superuser: "postgres"
patroni_superuser_password: "strongpassword"
patroni_replication_user: "replicator"
patroni_replication_password: "replpassword"
patroni_rewind_user: "rewinduser"
patroni_rewind_password: "rewindpassword"

# ============================
# REST API Patroni
# ============================
patroni_restapi_listen: "0.0.0.0:{{ patroni_restapi_port }}"

# ============================
# Списки хостов (переопределяются в inventory)
# ============================
etcd_members: []
postgres_nodes: []

# ============================
# Пути
# ============================
etcd_binary_install_dir: "/usr/local/bin"
patroni_venv_dir: "/opt/patroni-venv"
patroni_config_dir: "/etc/patroni"
etcd_config_dir: "/etc/etcd"
systemd_service_dir: "/etc/systemd/system"
postgresql_data_dir_debian: "/var/lib/postgresql/{{ postgresql_version }}/main"
postgresql_data_dir_redhat: "/var/lib/pgsql/{{ postgresql_version }}/data"

# Файлы для хранения учётных данных Nexus
nexus_apt_auth_file: "/etc/apt/auth.conf.d/nexus.conf"
nexus_pip_conf: "/etc/pip.conf"
```

---

2. vars/Debian.yml (Ubuntu 22.04 / 24.04)

```yaml
---
postgresql_packages:
  - "postgresql-{{ postgresql_version }}"
  - "postgresql-contrib-{{ postgresql_version }}"
  - "postgresql-client-{{ postgresql_version }}"

patroni_build_packages:
  - python3
  - python3-pip
  - python3-venv
  - python3-dev
  - libpq-dev
  - build-essential
  - ca-certificates
  - gnupg
  - wget
  - lsb-release

postgresql_bin_dir: "/usr/lib/postgresql/{{ postgresql_version }}/bin"
postgresql_service_name: "postgresql@{{ postgresql_version }}-main"
postgresql_data_dir: "{{ postgresql_data_dir_debian }}"
package_manager: apt
selinux_package: []

# APT-репозиторий PGDG через Nexus (без прямого доступа в интернет)
# Формат: <nexus_url>/repository/<repo>/<distribution>-pgdg <component>
postgresql_apt_repo: "deb [signed-by=/usr/share/keyrings/pgdg.gpg] {{ nexus_apt_pgdg_url }}/ {{ ansible_distribution_release }}-pgdg main"
```

---

3. vars/RedHat.yml (Oracle Linux 9)

```yaml
---
postgresql_packages:
  - "postgresql{{ postgresql_version }}"
  - "postgresql{{ postgresql_version }}-server"
  - "postgresql{{ postgresql_version }}-contrib"

patroni_build_packages:
  - python3
  - python3-pip
  - python3-devel
  - libpq-devel
  - gcc
  - make
  - python3-virtualenv

postgresql_bin_dir: "/usr/pgsql-{{ postgresql_version }}/bin"
postgresql_service_name: "postgresql-{{ postgresql_version }}"
postgresql_data_dir: "{{ postgresql_data_dir_redhat }}"
package_manager: dnf

selinux_package:
  - policycoreutils-python-utils
  - libselinux-python3

# YUM-репозиторий PGDG через Nexus
postgresql_yum_repo_file: "/etc/yum.repos.d/pgdg-nexus.repo"
```

---

4. tasks/install_etcd.yml — скачивание через Nexus с авторизацией

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

- name: Download etcd tarball from Nexus (with auth)
  ansible.builtin.get_url:
    url: "{{ nexus_etcd_download_url }}"
    dest: "/tmp/etcd-v{{ etcd_version }}-linux-amd64.tar.gz"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    validate_certs: "{{ nexus_validate_certs | default(true) }}"
    owner: root
    group: root
    mode: '0640'
  register: etcd_download

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

Формат URL в Nexus: …/repository/raw-etcd/v3.5.16/etcd-v3.5.16-linux-amd64.tar.gz. Администратор Nexus должен загрузить tarball в указанный путь raw-репозитория.

---

5. tasks/install_postgres.yml — установка через Nexus-прокси

```yaml
---
# =========================================================
# Debian / Ubuntu — APT прокси Nexus
# =========================================================
- name: Install APT prerequisites (Debian)
  ansible.builtin.apt:
    name:
      - ca-certificates
      - gnupg
      - wget
      - lsb-release
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Configure APT auth for Nexus (Debian)
  ansible.builtin.copy:
    dest: "{{ nexus_apt_auth_file }}"
    owner: root
    group: root
    mode: '0600'
    content: |
      machine {{ nexus_url | urlsplit('hostname') }}
      login {{ nexus_username }}
      password {{ nexus_password }}
  when: ansible_os_family == "Debian"

- name: Download PGDG signing key via Nexus (Debian)
  ansible.builtin.get_url:
    url: "{{ nexus_url }}/repository/{{ nexus_repo_apt_pgdg }}/ACCC4CF8.asc"
    dest: /tmp/ACCC4CF8.asc
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    owner: root
    group: root
    mode: '0644'
  when: ansible_os_family == "Debian"
  register: pgdg_key_download
  failed_when:
    - pgdg_key_download.failed
    - "'Status code was 404' not in pgdg_key_download.msg | default('')"

- name: Import PGDG signing key (Debian)
  ansible.builtin.command:
    cmd: "gpg --dearmor -o /usr/share/keyrings/pgdg.gpg /tmp/ACCC4CF8.asc"
    creates: /usr/share/keyrings/pgdg.gpg
  when: ansible_os_family == "Debian"

- name: Add PGDG repository via Nexus (Debian)
  ansible.builtin.apt_repository:
    repo: "{{ postgresql_apt_repo }}"
    state: present
    update_cache: yes
    filename: pgdg-nexus
  when: ansible_os_family == "Debian"

- name: Install PostgreSQL (Debian)
  ansible.builtin.apt:
    name: "{{ postgresql_packages }}"
    state: present
  when: ansible_os_family == "Debian"

# =========================================================
# RedHat / Oracle Linux 9 — YUM прокси Nexus
# =========================================================
- name: Deploy YUM repository file for PGDG via Nexus (RedHat)
  ansible.builtin.copy:
    dest: "{{ postgresql_yum_repo_file }}"
    owner: root
    group: root
    mode: '0644'
    content: |
      [pgdg-nexus]
      name=PGDG {{ postgresql_version }} via Nexus
      baseurl={{ nexus_yum_pgdg_url }}/{{ postgresql_version }}/x86_64/
      enabled=1
      gpgcheck=1
      gpgkey={{ nexus_yum_pgdg_url }}/RPM-GPG-KEY-PGDG
      username={{ nexus_username }}
      password={{ nexus_password }}
      sslverify={{ nexus_validate_certs | default(true) | lower }}
  when: ansible_os_family == "RedHat"

- name: Disable built-in PostgreSQL module (Oracle Linux 9)
  ansible.builtin.command:
    cmd: "dnf -qy module disable postgresql"
  changed_when: false
  when: ansible_os_family == "RedHat"

- name: Install PostgreSQL (RedHat)
  ansible.builtin.dnf:
    name: "{{ postgresql_packages }}"
    state: present
    disablerepo: "*"
    enablerepo: "pgdg-nexus,ol9_baseos_latest,ol9_appstream"
  when: ansible_os_family == "RedHat"

# =========================================================
# Общие задачи
# =========================================================
- name: Stop and disable default PostgreSQL service
  ansible.builtin.systemd:
    name: "{{ postgresql_service_name }}"
    state: stopped
    enabled: no
  ignore_errors: yes

- name: Ensure PostgreSQL data directory exists
  ansible.builtin.file:
    path: "{{ postgresql_data_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: '0700'
```

Важно про Oracle Linux 9: список enablerepo может отличаться в зависимости от имён репозиториев вашей системы (ol9_baseos_latest, ol9_appstream, ol9_UEKR7). Проверьте dnf repolist и при необходимости скорректируйте переменную.

---

6. tasks/install_patroni.yml — установка Patroni из Nexus PyPI

```yaml
---
- name: Install Python and build dependencies
  ansible.builtin.package:
    name: "{{ patroni_build_packages }}"
    state: present

- name: Configure pip to use Nexus PyPI proxy (global)
  ansible.builtin.copy:
    dest: "{{ nexus_pip_conf }}"
    owner: root
    group: root
    mode: '0644'
    content: |
      [global]
      index-url = {{ nexus_pypi_index_url }}
      trusted-host = {{ nexus_pypi_trusted_host }}
      # Nexus требует авторизации
      # Пароль хранится в netrc-файле ниже
  notify: restart patroni

- name: Create netrc file for pip authentication to Nexus (root)
  ansible.builtin.copy:
    dest: /root/.netrc
    owner: root
    group: root
    mode: '0600'
    content: |
      machine {{ nexus_pypi_trusted_host }}
      login {{ nexus_username }}
      password {{ nexus_password }}

- name: Create netrc file for postgres user (для установки pip от его имени)
  ansible.builtin.copy:
    dest: /var/lib/postgresql/.netrc
    owner: postgres
    group: postgres
    mode: '0600'
    content: |
      machine {{ nexus_pypi_trusted_host }}
      login {{ nexus_username }}
      password {{ nexus_password }}
  when: ansible_os_family == "Debian"

- name: Create netrc file for postgres user (RedHat)
  ansible.builtin.copy:
    dest: /var/lib/pgsql/.netrc
    owner: postgres
    group: postgres
    mode: '0600'
    content: |
      machine {{ nexus_pypi_trusted_host }}
      login {{ nexus_username }}
      password {{ nexus_password }}
  when: ansible_os_family == "RedHat"

- name: Upgrade pip using Nexus
  ansible.builtin.pip:
    name: pip
    state: latest
    executable: pip3
    extra_args: >-
      --index-url {{ nexus_pypi_index_url }}
      --trusted-host {{ nexus_pypi_trusted_host }}

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

- name: Install Patroni and dependencies inside virtualenv from Nexus
  ansible.builtin.pip:
    name:
      - "patroni[etcd]=={{ patroni_version }}"
      - "psycopg2-binary"
    virtualenv: "{{ patroni_venv_dir }}"
    virtualenv_python: python3
    extra_args: >-
      --index-url {{ nexus_pypi_index_url }}
      --trusted-host {{ nexus_pypi_trusted_host }}
  become_user: postgres

- name: Create Patroni configuration directory
  ansible.builtin.file:
    path: "{{ patroni_config_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: '0755'

- name: Create symlink to patronictl for convenience
  ansible.builtin.file:
    src: "{{ patroni_venv_dir }}/bin/patronictl"
    dest: "/usr/local/bin/patronictl"
    state: link
  ignore_errors: yes
```

---

7. Изменённый handlers/main.yml

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

- name: reload systemd
  ansible.builtin.systemd:
    daemon_reload: yes
```

---

8. Пример inventory

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
# Учётные данные Nexus (рекомендуется хранить в ansible-vault)
nexus_url: "https://nexus.example.com"
nexus_username: "ansible-reader"
nexus_password: "ChangeMe123!"
nexus_validate_certs: true

# Имена репозиториев в Nexus
nexus_repo_etcd_raw: "raw-etcd"
nexus_repo_pypi: "pypi-proxy"
nexus_repo_apt_pgdg: "apt-pgdg-proxy"
nexus_repo_yum_pgdg: "yum-pgdg-proxy"

# Кластер
etcd_members:
  - { name: node1,   ip: 192.168.1.11 }
  - { name: node2,   ip: 192.168.1.12 }
  - { name: witness, ip: 192.168.1.13 }

postgres_nodes:
  - node1
  - node2
```

---

9. Пример playbook

playbook.yml:

```yaml
---
- name: Deploy PostgreSQL 17 HA cluster with Patroni and etcd (via Nexus)
  hosts: all
  become: yes
  gather_facts: yes
  roles:
    - postgres-patroni-cluster
```

Запуск:

```bash
ansible-playbook -i inventory/hosts.yml playbook.yml
```

---

10. Что нужно подготовить в Nexus (однократно)

Ниже — список артефактов, которые администратор должен положить в Nexus, чтобы роль работала:

Репозиторий в Nexus Тип Содержимое
raw-etcd Raw (hosted) Каталог v3.5.16/etcd-v3.5.16-linux-amd64.tar.gz (скачан с github.com/etcd-io/etcd)
pypi-proxy PyPI (proxy) Прокси на https://pypi.org
apt-pgdg-proxy APT (proxy) Прокси на http://apt.postgresql.org/pub/repos/apt/
yum-pgdg-proxy YUM (proxy) Прокси на https://download.postgresql.org/pub/repos/yum/

Права пользователя ansible-reader — read-only доступ к этим четырём репозиториям.

Ключ PGDG для APT: файл ACCC4CF8.asc должен быть доступен по адресу
{{ nexus_url }}/repository/{{ nexus_repo_apt_pgdg }}/ACCC4CF8.asc — его нужно положить в raw-каталог того же APT-репозитория Nexus, либо использовать отдельный raw-репозиторий.

---

11. Хранение секретов

Рекомендуется зашифровать переменные Nexus через ansible-vault:

```bash
ansible-vault encrypt_string 'ChangeMe123!' --name 'nexus_password'
```

И использовать в group_vars/all.yml:

```yaml
nexus_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  3132333435363738...
```

Запуск:

```bash
ansible-playbook -i inventory/hosts.yml playbook.yml --ask-vault-pass
```

---

12. Итоговые изменения по сравнению с предыдущей версией

Область Было Стало
etcd get_url напрямую с github.com get_url с Nexus raw + basic auth
Patroni (pip) pip install из PyPI pip с index-url Nexus + .netrc
PostgreSQL APT apt.postgresql.org Nexus APT-прокси + /etc/apt/auth.conf.d/nexus.conf
PostgreSQL YUM download.postgresql.org Nexus YUM-прокси + .repo с username/password
Секреты отсутствовали nexus_username / nexus_password (ansible-vault)
Пакетные ключи скачивались с postgresql.org скачиваются через Nexus

Все интернет-запросы теперь идут только в локальный Nexus. Роль корректно работает на Oracle Linux 9, Ubuntu 22.04 и Ubuntu 24.04.


_______________________________________
_______________________________________
________________________________________
________________________________________

# Полная Ansible роль: PostgreSQL 17 + Patroni + etcd (2 ноды + свидетель)

Поддержка **Ubuntu 22.04, Ubuntu 24.04, Oracle Linux 9** с использованием **локального Nexus** для etcd, Patroni и PostgreSQL (со своим репозиторием и публичным ключом для каждой версии ОС).

---

## 📁 Структура роли

```
postgres-patroni-cluster/
├── defaults/
│   └── main.yml
├── vars/
│   ├── Debian.yml
│   └── RedHat.yml
├── meta/
│   └── main.yml
├── handlers/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── detect_pgdg_repo.yml
│   ├── install_etcd.yml
│   ├── install_postgres.yml
│   ├── install_patroni.yml
│   ├── configure_etcd.yml
│   ├── configure_patroni.yml
│   └── start_services.yml
└── templates/
    ├── etcd.conf.j2
    ├── etcd.service.j2
    ├── patroni.yml.j2
    └── patroni.service.j2
```

---

## 1. `defaults/main.yml`

```yaml
---
# ============================================================
# Версии компонентов
# ============================================================
postgresql_version: "17"
etcd_version: "3.5.16"
patroni_version: "4.0.4"

# ============================================================
# Nexus (уже настроен администратором)
# ============================================================
nexus_url: "https://nexus.example.com"
nexus_username: "ansible-reader"
nexus_password: "ChangeMe123!"
nexus_validate_certs: true

# ---------- Raw-репозитории (бинарники, wheels, ключи) ----------
nexus_repo_etcd_raw: "raw-etcd"
nexus_repo_python_raw: "raw-python-wheels"
nexus_repo_keys_raw: "raw-keys"

# ---------- Прокси-репозитории APT/YUM (по версии ОС) ----------
nexus_repo_apt_pgdg_jammy: "apt-pgdg-jammy"
nexus_repo_apt_pgdg_noble: "apt-pgdg-noble"
nexus_repo_yum_pgdg_el9:   "yum-pgdg-el9"

# ============================================================
# URL-производные
# ============================================================
nexus_hostname: "{{ nexus_url | urlsplit('hostname') }}"

nexus_etcd_download_url: >-
  {{ nexus_url }}/repository/{{ nexus_repo_etcd_raw }}/v{{ etcd_version }}/etcd-v{{ etcd_version }}-linux-amd64.tar.gz

nexus_patroni_find_links_url: >-
  {{ nexus_url }}/repository/{{ nexus_repo_python_raw }}/patroni/{{ patroni_version }}/

# ============================================================
# Параметры кластера
# ============================================================
cluster_name: "pg-cluster"
etcd_initial_cluster_token: "etcd-cluster-token"
etcd_data_dir: "/var/lib/etcd"

# ============================================================
# Порты
# ============================================================
etcd_client_port: 2379
etcd_peer_port: 2380
postgresql_port: 5432
patroni_restapi_port: 8008

# ============================================================
# Настройки etcd
# ============================================================
etcd_heartbeat_interval: 1000
etcd_election_timeout: 5000
etcd_initial_cluster_state: "new"

# ============================================================
# Учётные данные PostgreSQL / Patroni
# ============================================================
patroni_superuser: "postgres"
patroni_superuser_password: "strongpassword"
patroni_replication_user: "replicator"
patroni_replication_password: "replpassword"
patroni_rewind_user: "rewinduser"
patroni_rewind_password: "rewindpassword"

# ============================================================
# REST API Patroni
# ============================================================
patroni_restapi_listen: "0.0.0.0:{{ patroni_restapi_port }}"

# ============================================================
# Списки хостов (переопределяются в inventory)
# ============================================================
etcd_members: []
postgres_nodes: []

# ============================================================
# Пути
# ============================================================
etcd_binary_install_dir: "/usr/local/bin"
patroni_venv_dir: "/opt/patroni-venv"
patroni_config_dir: "/etc/patroni"
etcd_config_dir: "/etc/etcd"
systemd_service_dir: "/etc/systemd/system"

postgresql_data_dir_debian: "/var/lib/postgresql/{{ postgresql_version }}/main"
postgresql_data_dir_redhat: "/var/lib/pgsql/{{ postgresql_version }}/data"
postgres_home_debian: "/var/lib/postgresql"
postgres_home_redhat: "/var/lib/pgsql"

# ============================================================
# Файлы авторизации
# ============================================================
nexus_apt_auth_file: "/etc/apt/auth.conf.d/nexus.conf"

# ============================================================
# Факты, устанавливаемые в detect_pgdg_repo.yml
# ============================================================
current_pgdg_apt_url: ""
current_pgdg_apt_component: ""
current_pgdg_yum_url: ""
current_pgdg_key_file: ""
```

---

## 2. `vars/Debian.yml` (Ubuntu 22.04 / 24.04)

```yaml
---
postgresql_packages:
  - "postgresql-{{ postgresql_version }}"
  - "postgresql-contrib-{{ postgresql_version }}"
  - "postgresql-client-{{ postgresql_version }}"

patroni_build_packages:
  - python3
  - python3-pip
  - python3-venv
  - python3-dev
  - libpq-dev
  - build-essential
  - ca-certificates
  - gnupg
  - wget
  - lsb-release

postgresql_bin_dir: "/usr/lib/postgresql/{{ postgresql_version }}/bin"
postgresql_service_name: "postgresql@{{ postgresql_version }}-main"
postgresql_data_dir: "{{ postgresql_data_dir_debian }}"
postgres_home: "{{ postgres_home_debian }}"
package_manager: apt
selinux_package: []
```

---

## 3. `vars/RedHat.yml` (Oracle Linux 9)

```yaml
---
postgresql_packages:
  - "postgresql{{ postgresql_version }}"
  - "postgresql{{ postgresql_version }}-server"
  - "postgresql{{ postgresql_version }}-contrib"

patroni_build_packages:
  - python3
  - python3-pip
  - python3-devel
  - libpq-devel
  - gcc
  - make
  - python3-virtualenv
  - ca-certificates

postgresql_bin_dir: "/usr/pgsql-{{ postgresql_version }}/bin"
postgresql_service_name: "postgresql-{{ postgresql_version }}"
postgresql_data_dir: "{{ postgresql_data_dir_redhat }}"
postgres_home: "{{ postgres_home_redhat }}"
package_manager: dnf
postgresql_yum_repo_file: "/etc/yum.repos.d/pgdg-nexus.repo"

selinux_package:
  - policycoreutils-python-utils
  - libselinux-python3
```

---

## 4. `meta/main.yml`

```yaml
---
galaxy_info:
  author: "Your Name"
  description: >
    Deploy PostgreSQL 17 HA cluster with Patroni and etcd
    (2 DB nodes + witness) on Oracle Linux 9, Ubuntu 22.04/24.04,
    using a local Nexus repository for all artifacts.
  license: "MIT"
  min_ansible_version: "2.12"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
    - name: EL
      versions:
        - "9"
  galaxy_tags:
    - postgresql
    - patroni
    - etcd
    - highavailability
    - cluster
    - nexus
dependencies: []
```

---

## 5. `handlers/main.yml`

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

- name: reload systemd
  ansible.builtin.systemd:
    daemon_reload: yes
```

---

## 6. `tasks/main.yml`

```yaml
---
- name: Load OS-specific variables
  ansible.builtin.include_vars: "{{ ansible_os_family }}.yml"

- name: Fail if OS is not supported
  ansible.builtin.fail:
    msg: "Unsupported OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
  when: >
    not (
      (ansible_distribution == 'Ubuntu' and ansible_distribution_release in ['jammy', 'noble'])
      or
      (ansible_os_family == 'RedHat' and ansible_distribution_major_version == '9')
    )

- name: Detect PGDG repository settings for current OS version
  ansible.builtin.include_tasks: detect_pgdg_repo.yml

- name: Install etcd on all nodes
  ansible.builtin.include_tasks: install_etcd.yml
  when: inventory_hostname in groups['etcd_cluster']

- name: Install PostgreSQL on database nodes
  ansible.builtin.include_tasks: install_postgres.yml
  when: inventory_hostname in groups['postgres_cluster']

- name: Install Patroni on database nodes
  ansible.builtin.include_tasks: install_patroni.yml
  when: inventory_hostname in groups['postgres_cluster']

- name: Configure etcd on all nodes
  ansible.builtin.include_tasks: configure_etcd.yml
  when: inventory_hostname in groups['etcd_cluster']

- name: Configure Patroni on database nodes
  ansible.builtin.include_tasks: configure_patroni.yml
  when: inventory_hostname in groups['postgres_cluster']

- name: Start services
  ansible.builtin.include_tasks: start_services.yml
```

---

## 7. `tasks/detect_pgdg_repo.yml`

```yaml
---
# =========================================================
# Выбор репозитория PostgreSQL в Nexus в зависимости от версии ОС
# Устанавливаются факты:
#   - current_pgdg_apt_url       (только Debian family)
#   - current_pgdg_apt_component (только Debian family)
#   - current_pgdg_yum_url       (только RedHat family)
#   - current_pgdg_key_file      (имя файла ключа PGDG)
# =========================================================

- name: Detect PGDG repository for Ubuntu 22.04 (jammy)
  ansible.builtin.set_fact:
    current_pgdg_apt_url: "{{ nexus_url }}/repository/{{ nexus_repo_apt_pgdg_jammy }}"
    current_pgdg_apt_component: "jammy-pgdg"
    current_pgdg_key_file: "ACCC4CF8.asc"
  when:
    - ansible_distribution == "Ubuntu"
    - ansible_distribution_release == "jammy"

- name: Detect PGDG repository for Ubuntu 24.04 (noble)
  ansible.builtin.set_fact:
    current_pgdg_apt_url: "{{ nexus_url }}/repository/{{ nexus_repo_apt_pgdg_noble }}"
    current_pgdg_apt_component: "noble-pgdg"
    current_pgdg_key_file: "ACCC4CF8.asc"
  when:
    - ansible_distribution == "Ubuntu"
    - ansible_distribution_release == "noble"

- name: Detect PGDG repository for Oracle Linux 9
  ansible.builtin.set_fact:
    current_pgdg_yum_url: "{{ nexus_url }}/repository/{{ nexus_repo_yum_pgdg_el9 }}"
    current_pgdg_key_file: "RPM-GPG-KEY-PGDG-{{ postgresql_version }}"
  when:
    - ansible_os_family == "RedHat"
    - ansible_distribution_major_version == "9"

- name: Show detected PGDG repository
  ansible.builtin.debug:
    msg: >-
      {{ (ansible_os_family == 'Debian')
         | ternary('APT repo: ' ~ current_pgdg_apt_url ~ ' | component: ' ~ current_pgdg_apt_component,
                   'YUM repo: ' ~ current_pgdg_yum_url) }}
      | key file: {{ current_pgdg_key_file }}
```

---

## 8. `tasks/install_etcd.yml`

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

- name: Download etcd tarball from Nexus (with auth)
  ansible.builtin.get_url:
    url: "{{ nexus_etcd_download_url }}"
    dest: "/tmp/etcd-v{{ etcd_version }}-linux-amd64.tar.gz"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    validate_certs: "{{ nexus_validate_certs }}"
    owner: root
    group: root
    mode: '0640'

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

## 9. `tasks/install_postgres.yml`

```yaml
---
# =========================================================
# Debian / Ubuntu — APT прокси Nexus
# =========================================================
- name: Install APT prerequisites (Debian)
  ansible.builtin.apt:
    name:
      - ca-certificates
      - gnupg
      - wget
      - lsb-release
    state: present
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Ensure APT auth directory exists (Debian)
  ansible.builtin.file:
    path: "{{ nexus_apt_auth_file | dirname }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  when: ansible_os_family == "Debian"

- name: Configure APT auth for Nexus (Debian)
  ansible.builtin.copy:
    dest: "{{ nexus_apt_auth_file }}"
    owner: root
    group: root
    mode: '0600'
    content: |
      machine {{ nexus_hostname }}
      login {{ nexus_username }}
      password {{ nexus_password }}
  when: ansible_os_family == "Debian"

- name: Download PGDG signing key from Nexus (Debian)
  ansible.builtin.get_url:
    url: "{{ nexus_url }}/repository/{{ nexus_repo_keys_raw }}/{{ current_pgdg_key_file }}"
    dest: "/tmp/{{ current_pgdg_key_file }}"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    validate_certs: "{{ nexus_validate_certs }}"
    owner: root
    group: root
    mode: '0644'
  when: ansible_os_family == "Debian"

- name: Import PGDG signing key (Debian)
  ansible.builtin.shell:
    cmd: "gpg --dearmor -o /usr/share/keyrings/pgdg.gpg /tmp/{{ current_pgdg_key_file }}"
    creates: /usr/share/keyrings/pgdg.gpg
  when: ansible_os_family == "Debian"

- name: Add PGDG repository via Nexus (Debian)
  ansible.builtin.apt_repository:
    repo: >-
      deb [signed-by=/usr/share/keyrings/pgdg.gpg]
      {{ current_pgdg_apt_url }}/
      {{ current_pgdg_apt_component }} main
    state: present
    update_cache: yes
    filename: pgdg-nexus
  when: ansible_os_family == "Debian"

- name: Install PostgreSQL (Debian)
  ansible.builtin.apt:
    name: "{{ postgresql_packages }}"
    state: present
  when: ansible_os_family == "Debian"

# =========================================================
# RedHat / Oracle Linux 9 — YUM прокси Nexus
# =========================================================
- name: Download PGDG RPM signing key from Nexus (RedHat)
  ansible.builtin.get_url:
    url: "{{ nexus_url }}/repository/{{ nexus_repo_keys_raw }}/{{ current_pgdg_key_file }}"
    dest: "/etc/pki/rpm-gpg/{{ current_pgdg_key_file }}"
    url_username: "{{ nexus_username }}"
    url_password: "{{ nexus_password }}"
    force_basic_auth: yes
    validate_certs: "{{ nexus_validate_certs }}"
    owner: root
    group: root
    mode: '0644'
  when: ansible_os_family == "RedHat"

- name: Deploy YUM repo file for PGDG via Nexus (RedHat)
  ansible.builtin.copy:
    dest: "{{ postgresql_yum_repo_file }}"
    owner: root
    group: root
    mode: '0644'
    content: |
      [pgdg-nexus]
      name=PGDG {{ postgresql_version }} via Nexus (OL9)
      baseurl={{ current_pgdg_yum_url }}/{{ postgresql_version }}/x86_64/
      enabled=1
      gpgcheck=1
      gpgkey=file:///etc/pki/rpm-gpg/{{ current_pgdg_key_file }}
      username={{ nexus_username }}
      password={{ nexus_password }}
      sslverify={{ nexus_validate_certs | lower }}
  when: ansible_os_family == "RedHat"

- name: Disable built-in PostgreSQL module (Oracle Linux 9)
  ansible.builtin.command:
    cmd: "dnf -qy module disable postgresql"
  changed_when: false
  when: ansible_os_family == "RedHat"

- name: Install PostgreSQL (RedHat)
  ansible.builtin.dnf:
    name: "{{ postgresql_packages }}"
    state: present
  when: ansible_os_family == "RedHat"

# =========================================================
# Общие задачи
# =========================================================
- name: Stop and disable default PostgreSQL service
  ansible.builtin.systemd:
    name: "{{ postgresql_service_name }}"
    state: stopped
    enabled: no
  ignore_errors: yes

- name: Ensure PostgreSQL data directory exists
  ansible.builtin.file:
    path: "{{ postgresql_data_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: '0700'
```

---

## 10. `tasks/install_patroni.yml`

```yaml
---
- name: Install Python and build dependencies
  ansible.builtin.package:
    name: "{{ patroni_build_packages }}"
    state: present

# ---------- Авторизация pip в Nexus через .netrc ----------
- name: Create netrc file for root (Nexus auth)
  ansible.builtin.copy:
    dest: /root/.netrc
    owner: root
    group: root
    mode: '0600'
    content: |
      machine {{ nexus_hostname }}
      login {{ nexus_username }}
      password {{ nexus_password }}

- name: Create netrc file for postgres user (Nexus auth)
  ansible.builtin.copy:
    dest: "{{ postgres_home }}/.netrc"
    owner: postgres
    group: postgres
    mode: '0600'
    content: |
      machine {{ nexus_hostname }}
      login {{ nexus_username }}
      password {{ nexus_password }}

# ---------- Создание venv ----------
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

# ---------- Установка Patroni из Nexus wheels (offline) ----------
- name: Install Patroni from Nexus wheels (offline mode)
  ansible.builtin.pip:
    name:
      - "patroni[etcd]=={{ patroni_version }}"
    virtualenv: "{{ patroni_venv_dir }}"
    virtualenv_python: python3
    extra_args: >-
      --no-index
      --find-links={{ nexus_patroni_find_links_url }}
      --trusted-host {{ nexus_hostname }}
  become_user: postgres

- name: Ensure Patroni config directory exists
  ansible.builtin.file:
    path: "{{ patroni_config_dir }}"
    state: directory
    owner: postgres
    group: postgres
    mode: '0755'

- name: Create symlink to patronictl for convenience
  ansible.builtin.file:
    src: "{{ patroni_venv_dir }}/bin/patronictl"
    dest: "/usr/local/bin/patronictl"
    state: link
  ignore_errors: yes
```

---

## 11. `tasks/configure_etcd.yml`

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

## 12. `tasks/configure_patroni.yml`

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

## 13. `tasks/start_services.yml`

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
  when:
    - inventory_hostname in groups['etcd_cluster']
    - inventory_hostname == groups['etcd_cluster'][0]

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
  when:
    - inventory_hostname in groups['postgres_cluster']
    - inventory_hostname == groups['postgres_cluster'][0]

- name: Show cluster status
  ansible.builtin.debug:
    msg: "{{ patroni_status.stdout_lines }}"
  when:
    - inventory_hostname in groups['postgres_cluster']
    - inventory_hostname == groups['postgres_cluster'][0]
    - patroni_status.stdout_lines is defined
```

---

## 14. `templates/etcd.conf.j2`

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

---

## 15. `templates/etcd.service.j2`

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

---

## 16. `templates/patroni.yml.j2`

```yaml
scope: {{ cluster_name }}
namespace: /{{ cluster_name }}/
name: {{ inventory_hostname }}

restapi:
  listen: {{ patroni_restapi_listen }}
  connect_address: {{ ansible_default_ipv4.address }}:{{ patroni_restapi_port }}

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
  bin_dir: {{ postgresql_bin_dir }}
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

---

## 17. `templates/patroni.service.j2`

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

## 18. Пример `inventory/hosts.yml`

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

## 19. Пример `inventory/group_vars/all.yml`

```yaml
# ============ Nexus ============
nexus_url: "https://nexus.example.com"
nexus_username: "ansible-reader"
nexus_password: "ChangeMe123!"
nexus_validate_certs: true

# Raw-репозитории
nexus_repo_etcd_raw: "raw-etcd"
nexus_repo_python_raw: "raw-python-wheels"
nexus_repo_keys_raw: "raw-keys"

# APT/YUM прокси по версии ОС
nexus_repo_apt_pgdg_jammy: "apt-pgdg-jammy"
nexus_repo_apt_pgdg_noble: "apt-pgdg-noble"
nexus_repo_yum_pgdg_el9:   "yum-pgdg-el9"

# ============ Кластер ============
etcd_members:
  - { name: node1,   ip: 192.168.1.11 }
  - { name: node2,   ip: 192.168.1.12 }
  - { name: witness, ip: 192.168.1.13 }

postgres_nodes:
  - node1
  - node2

# ============ Секреты PostgreSQL/Patroni ============
patroni_superuser_password: "strongpassword"
patroni_replication_password: "replpassword"
patroni_rewind_password: "rewindpassword"
```

## 20. Пример `playbook.yml`

```yaml
---
- name: Deploy PostgreSQL 17 HA cluster (Patroni + etcd) via Nexus
  hosts: all
  become: yes
  gather_facts: yes
  roles:
    - postgres-patroni-cluster
```

Запуск:

```bash
ansible-playbook -i inventory/hosts.yml playbook.yml --ask-vault-pass
```

---

## 21. Что администратор должен подготовить в Nexus (однократно)

| Репозиторий Nexus | Тип | Что положить | Путь внутри |
|---|---|---|---|
| `raw-etcd` | Raw (hosted) | `etcd-v3.5.16-linux-amd64.tar.gz` | `v3.5.16/etcd-v3.5.16-linux-amd64.tar.gz` |
| `raw-python-wheels` | Raw (hosted) | все `.whl` (Patroni + зависимости) | `patroni/4.0.4/*.whl` |
| `raw-keys` | Raw (hosted) | `ACCC4CF8.asc`, `RPM-GPG-KEY-PGDG-17` | корень |
| `apt-pgdg-jammy` | APT proxy | proxy → `http://apt.postgresql.org/pub/repos/apt/` | — |
| `apt-pgdg-noble` | APT proxy | proxy → `http://apt.postgresql.org/pub/repos/apt/` | — |
| `yum-pgdg-el9` | YUM proxy | proxy → `https://download.postgresql.org/pub/repos/yum/` | — |

**Подготовка wheels для raw-python-wheels** (выполняется на машине с интернетом и Python той же версии/архитектуры, что и целевые хосты):

```bash
mkdir -p wheels/patroni/4.0.4
pip download "patroni[etcd]==4.0.4" psycopg2-binary \
    --dest wheels/patroni/4.0.4 \
    --only-binary=:all: \
    --python-version 3.10 --platform manylinux2014_x86_64
# загрузить содержимое wheels/patroni/4.0.4/*.whl в Nexus в путь patroni/4.0.4/
```

Пользователю `ansible-reader` достаточно **read-only** доступа к этим репозиториям.

---

## 22. Схема работы

```
┌───────────────────────────────────────────────────────────────────────┐
│ Ansible                                                               │
│   ├── etcd      → get_url → Nexus raw-etcd            (basic auth)    │
│   ├── Patroni   → pip     → Nexus raw-python-wheels   (.netrc)        │
│   ├── PG key    → get_url → Nexus raw-keys            (basic auth)    │
│   └── PG pkg    → apt/dnf → Nexus apt-pgdg-*/yum-pgdg-el9 (auth)      │
└───────────────────────────────────────────────────────────────────────┘
        │
        ▼
   Ubuntu 22.04   → apt-pgdg-jammy
   Ubuntu 24.04   → apt-pgdg-noble
   Oracle Linux 9 → yum-pgdg-el9
```

Выбор репозитория PostgreSQL производится автоматически в задаче `tasks/detect_pgdg_repo.yml` на основе `ansible_distribution` и `ansible_distribution_release` / `ansible_distribution_major_version`. Все интернет-запросы идут только в локальный Nexus. Публичный ключ PGDG для каждой версии ОС скачивается из Nexus raw-репозитория с basic auth.

Роль готова к запуску.

_______________________________
_______________________________
_______________________________
_______________________________


