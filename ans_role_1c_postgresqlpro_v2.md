# Роль Ansible для настройки PostgreSQL Pro для сервера 1С:Предприятие

Вот пример роли Ansible для настройки PostgreSQL Pro (Postgres Pro) для работы с 1С:Предприятие.

## Структура роли

```
roles/postgrespro_1c/
├── defaults
│   └── main.yml
├── files
│   ├── postgrespro-1c.repo
│   └── tuning.conf
├── handlers
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── pg_hba.conf.j2
└── vars
    └── main.yml
```

## Содержание файлов

### defaults/main.yml

```yaml
---
# Версия Postgres Pro для 1С
postgrespro_1c_version: "15"

# Параметры кластера
postgrespro_1c_cluster_name: "main"
postgrespro_1c_port: "5432"
postgrespro_1c_locale: "ru_RU.UTF-8"
postgrespro_1c_encoding: "UTF8"

# Пользователь и база для 1С
postgrespro_1c_user: "user1c"
postgrespro_1c_password: "password1c"
postgrespro_1c_database: "base1c"

# Параметры настройки
postgrespro_1c_shared_buffers: "4GB"
postgrespro_1c_work_mem: "32MB"
postgrespro_1c_maintenance_work_mem: "1GB"
postgrespro_1c_effective_cache_size: "12GB"
```

### vars/main.yml

```yaml
---
# Репозиторий Postgres Pro
postgrespro_1c_repo_url: "https://repo.postgrespro.ru/1c-{{ postgrespro_1c_version }}/keys/postgrespro-1c-{{ postgrespro_1c_version }}.repo"

# Пакеты для установки
postgrespro_1c_packages:
  - "postgrespro-1c-{{ postgrespro_1c_version }}-server"
  - "postgrespro-1c-{{ postgrespro_1c_version }}-contrib"
  - "postgrespro-1c-{{ postgrespro_1c_version }}-libs"
  - "postgrespro-1c-{{ postgrespro_1c_version }}-debuginfo"
```

### files/postgrespro-1c.repo

```
[postgrespro-1c]
name=Postgres Pro 1C $releasever - $basearch
baseurl=https://repo.postgrespro.ru/1c-{{ postgrespro_1c_version }}/redhat/rhel-$releasever-$basearch
enabled=1
gpgcheck=1
gpgkey=https://repo.postgrespro.ru/1c-{{ postgrespro_1c_version }}/keys/PGPRO-1C-{{ postgrespro_1c_version }}-GPG-KEY
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  package:
    name:
      - wget
      - epel-release
    state: present

- name: Добавление репозитория Postgres Pro для 1С
  get_url:
    url: "{{ postgrespro_1c_repo_url }}"
    dest: /etc/yum.repos.d/postgrespro-1c.repo
    mode: '0644'

- name: Установка Postgres Pro для 1С
  package:
    name: "{{ postgrespro_1c_packages }}"
    state: present

- name: Инициализация кластера Postgres Pro
  command: >
    /opt/pgpro/{{ postgrespro_1c_version }}/bin/pg-setup initdb
    {{ postgrespro_1c_cluster_name }}
    --locale={{ postgrespro_1c_locale }}
    --encoding={{ postgrespro_1c_encoding }}
  when: not ansible_check_mode
  register: initdb_result
  changed_when: "'already exists' not in initdb_result.stderr"

- name: Копирование настроек производительности
  copy:
    src: tuning.conf
    dest: /etc/postgresql/{{ postgrespro_1c_cluster_name }}/tuning.conf
    owner: postgres
    group: postgres
    mode: '0644'
  notify: restart postgresql

- name: Настройка pg_hba.conf
  template:
    src: pg_hba.conf.j2
    dest: /etc/postgresql/{{ postgrespro_1c_cluster_name }}/pg_hba.conf
    owner: postgres
    group: postgres
    mode: '0640'
  notify: restart postgresql

- name: Включение и запуск службы Postgres Pro
  service:
    name: "postgresql-{{ postgrespro_1c_cluster_name }}"
    enabled: yes
    state: started

- name: Создание пользователя для 1С
  postgresql_user:
    login_user: postgres
    login_host: localhost
    name: "{{ postgrespro_1c_user }}"
    password: "{{ postgrespro_1c_password }}"
    role_attr_flags: "CREATEDB,CREATEROLE,LOGIN"
    state: present

- name: Создание базы данных для 1С
  postgresql_db:
    login_user: postgres
    login_host: localhost
    name: "{{ postgrespro_1c_database }}"
    owner: "{{ postgrespro_1c_user }}"
    encoding: "{{ postgrespro_1c_encoding }}"
    lc_ctype: "{{ postgrespro_1c_locale }}"
    lc_collate: "{{ postgrespro_1c_locale }}"
    state: present
```

### templates/pg_hba.conf.j2

```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer

# IPv4 local connections:
host    all             all             127.0.0.1/32            md5

# IPv6 local connections:
host    all             all             ::1/128                 md5

# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5

# 1C connections
host    {{ postgrespro_1c_database }}  {{ postgrespro_1c_user }}  0.0.0.0/0               md5
```

### files/tuning.conf

```
# Настройки производительности для 1С
max_connections = 100
shared_buffers = {{ postgrespro_1c_shared_buffers }}
work_mem = {{ postgrespro_1c_work_mem }}
maintenance_work_mem = {{ postgrespro_1c_maintenance_work_mem }}
effective_cache_size = {{ postgrespro_1c_effective_cache_size }}

# Оптимизации для 1С
random_page_cost = 1.5
effective_io_concurrency = 200
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.05
autovacuum_max_workers = 3
autovacuum_naptime = 30s
```

### handlers/main.yml

```yaml
---
- name: restart postgresql
  service:
    name: "postgresql-{{ postgrespro_1c_cluster_name }}"
    state: restarted
```

## Использование роли

Пример playbook для использования этой роли:

```yaml
- hosts: 1c_servers
  become: yes
  roles:
    - role: postgrespro_1c
      vars:
        postgrespro_1c_user: "usr1cprod"
        postgrespro_1c_password: "Str0ngP@ssw0rd"
        postgrespro_1c_database: "prod1cbase"
```

## Дополнительные рекомендации

1. Для продакшн-среды обязательно измените пароли в переменных.
2. Настройте резервное копирование баз данных.
3. Для больших баз данных 1С может потребоваться дополнительная настройка параметров PostgreSQL.
4. Убедитесь, что сервер имеет достаточные ресурсы (RAM, CPU) для работы с вашей нагрузкой 1С.

Эта роль обеспечивает базовую настройку PostgreSQL Pro для работы с 1С:Предприятие. Вы можете дополнительно настроить её под свои конкретные требования.