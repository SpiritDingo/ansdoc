# Ansible роль для настройки PostgreSQL Pro для сервера 1С:Предприятия

Создадим роль для установки и настройки PostgreSQL Pro (от Postgres Pro) - оптимизированной версии PostgreSQL для работы с 1С:Предприятием.

## Структура роли

```
roles/postgresqlpro_1c/
├── tasks/
│   ├── main.yml
│   ├── install.yml
│   ├── configure.yml
│   └── setup_1c.yml
├── templates/
│   ├── postgresql.conf.j2
│   ├── pg_hba.conf.j2
│   └── create_1c_db.sql.j2
├── vars/
│   └── main.yml
└── handlers/
    └── main.yml
```

## Файлы роли

### tasks/main.yml
```yaml
- name: Include installation tasks
  include: install.yml
  tags: install

- name: Include configuration tasks
  include: configure.yml
  tags: configure

- name: Include 1C specific setup tasks
  include: setup_1c.yml
  tags: setup_1c
```

### tasks/install.yml
```yaml
- name: Add Postgres Pro repository key
  apt_key:
    url: "https://repo.postgrespro.ru/pgpro-14/keys/GPG-KEY-POSTGRESPRO"
    state: present
  when: ansible_os_family == 'Debian'

- name: Add Postgres Pro repository (Debian/Ubuntu)
  apt_repository:
    repo: "deb https://repo.postgrespro.ru/pgpro-14/debian {{ ansible_distribution_release }} main"
    state: present
    filename: "postgrespro-1c"
  when: ansible_os_family == 'Debian'

- name: Add Postgres Pro repository (RHEL/CentOS)
  yum_repository:
    name: "postgrespro-1c"
    description: "Postgres Pro 1C repository"
    baseurl: "https://repo.postgrespro.ru/pgpro-14/centos/{{ ansible_distribution_major_version }}-{{ ansible_architecture }}"
    gpgcheck: yes
    gpgkey: "https://repo.postgrespro.ru/pgpro-14/keys/GPG-KEY-POSTGRESPRO"
    enabled: yes
  when: ansible_os_family == 'RedHat'

- name: Install Postgres Pro 1C package
  package:
    name: "{{ postgrespro_package }}"
    state: present
    update_cache: yes

- name: Install additional packages
  package:
    name: "{{ item }}"
    state: present
  loop: "{{ postgrespro_additional_packages }}"
```

### tasks/configure.yml
```yaml
- name: Create PostgreSQL data directory
  file:
    path: "{{ postgresql_data_dir }}"
    state: directory
    owner: "{{ postgresql_user }}"
    group: "{{ postgresql_group }}"
    mode: '0700'

- name: Configure postgresql.conf
  template:
    src: postgresql.conf.j2
    dest: "{{ postgresql_config_path }}/postgresql.conf"
    owner: "{{ postgresql_user }}"
    group: "{{ postgresql_group }}"
    mode: '0644'
  notify: restart postgresql

- name: Configure pg_hba.conf
  template:
    src: pg_hba.conf.j2
    dest: "{{ postgresql_config_path }}/pg_hba.conf"
    owner: "{{ postgresql_user }}"
    group: "{{ postgresql_group }}"
    mode: '0640'
  notify: reload postgresql

- name: Ensure PostgreSQL service is running and enabled
  service:
    name: "{{ postgresql_service_name }}"
    state: started
    enabled: yes
```

### tasks/setup_1c.yml
```yaml
- name: Create 1C database user
  postgresql_user:
    name: "{{ db_1c_user }}"
    password: "{{ db_1c_password }}"
    role_attr_flags: "CREATEDB,NOLOGIN"
    state: present

- name: Create SQL script for 1C database
  template:
    src: create_1c_db.sql.j2
    dest: "/tmp/create_1c_db.sql"
    mode: '0600'

- name: Create 1C database
  postgresql_db:
    name: "{{ db_1c_name }}"
    owner: "{{ db_1c_user }}"
    encoding: "WIN1251"
    lc_collate: "ru_RU.UTF-8"
    lc_ctype: "ru_RU.UTF-8"
    template: "template0"
    state: present

- name: Apply additional 1C specific settings
  become_user: "{{ postgresql_user }}"
  command: "psql -d {{ db_1c_name }} -f /tmp/create_1c_db.sql"
  changed_when: false
```

### templates/postgresql.conf.j2
```ini
# Основные параметры
listen_addresses = '*'
port = {{ postgresql_port }}
max_connections = 200
superuser_reserved_connections = 3

# Настройки памяти
shared_buffers = {{ (ansible_memtotal_mb * 0.25)|int }}MB
work_mem = 16MB
maintenance_work_mem = 128MB
effective_cache_size = {{ (ansible_memtotal_mb * 0.5)|int }}MB

# Настройки WAL
wal_level = replica
synchronous_commit = off
wal_buffers = 16MB
checkpoint_completion_target = 0.9

# Настройки для 1С
random_page_cost = 1.5
effective_io_concurrency = 2
max_wal_size = 4GB
min_wal_size = 1GB

# Логирование
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_truncate_on_rotation = on
log_rotation_age = 1d
log_rotation_size = 0
log_min_duration_statement = 1000
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
log_temp_files = 0
log_line_prefix = '%m [%p] %q%u@%d '
log_timezone = 'Europe/Moscow'

# Время ожидания
deadlock_timeout = 3s
lock_timeout = 0
statement_timeout = 0
idle_in_transaction_session_timeout = 0

# Настройки для 1С
default_statistics_target = 500
constraint_exclusion = on
```

### templates/pg_hba.conf.j2
```ini
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer

# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256

# IPv6 local connections:
host    all             all             ::1/128                 scram-sha-256

# Allow 1C servers to connect
host    all             {{ db_1c_user }}       {{ 1c_servers_network }}       scram-sha-256

# Allow replication connections from 1C cluster servers
host    replication     replicator      {{ 1c_cluster_network }}       scram-sha-256
```

### templates/create_1c_db.sql.j2
```sql
-- Оптимизация для 1С
ALTER SYSTEM SET synchronous_commit TO 'off';
ALTER SYSTEM SET checkpoint_completion_target TO '0.9';
ALTER SYSTEM SET random_page_cost TO '1.5';
ALTER SYSTEM SET effective_io_concurrency TO '2';

-- Создание расширений для 1С
CREATE EXTENSION IF NOT EXISTS plpgsql;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Настройки для работы с 1С
ALTER DATABASE {{ db_1c_name }} SET default_text_search_config TO 'russian';
ALTER DATABASE {{ db_1c_name }} SET lc_messages TO 'ru_RU.UTF-8';
ALTER DATABASE {{ db_1c_name }} SET lc_monetary TO 'ru_RU.UTF-8';
ALTER DATABASE {{ db_1c_name }} SET lc_numeric TO 'ru_RU.UTF-8';
ALTER DATABASE {{ db_1c_name }} SET lc_time TO 'ru_RU.UTF-8';
ALTER DATABASE {{ db_1c_name }} SET intervalstyle TO 'postgres';
ALTER DATABASE {{ db_1c_name }} SET timezone TO 'Europe/Moscow';

-- Права для пользователя 1С
GRANT ALL PRIVILEGES ON DATABASE {{ db_1c_name }} TO {{ db_1c_user }};
```

### vars/main.yml
```yaml
# Параметры установки
postgrespro_package: "postgrespro-1c-14"
postgrespro_additional_packages:
  - "postgrespro-1c-14-contrib"
  - "postgrespro-1c-14-server-dev"
  - "postgrespro-1c-14-libs"

# Параметры службы
postgresql_service_name: "postgrespro-1c-14"
postgresql_user: "postgres"
postgresql_group: "postgres"
postgresql_port: 5432

# Пути
postgresql_data_dir: "/var/lib/pgpro/1c-14/data"
postgresql_config_path: "/var/lib/pgpro/1c-14/data"

# Параметры БД для 1С
db_1c_name: "db1c"
db_1c_user: "user1c"
db_1c_password: "changeme"

# Сети для доступа
1c_servers_network: "192.168.1.0/24"
1c_cluster_network: "192.168.1.0/24"
```

### handlers/main.yml
```yaml
- name: restart postgresql
  service:
    name: "{{ postgresql_service_name }}"
    state: restarted

- name: reload postgresql
  service:
    name: "{{ postgresql_service_name }}"
    state: reloaded
```

## Пример использования роли в playbook

```yaml
- name: Configure Postgres Pro for 1C
  hosts: db_servers
  become: yes
  vars:
    db_1c_password: "{{ vault_db_1c_password }}"  # Используйте Ansible Vault для безопасности
  roles:
    - role: postgresqlpro_1c
```

## Дополнительные рекомендации

1. **Безопасность паролей**: Всегда используйте Ansible Vault для хранения паролей.
2. **Резервное копирование**: Добавьте задачи для настройки регулярного резервного копирования.
3. **Мониторинг**: Настройте мониторинг PostgreSQL