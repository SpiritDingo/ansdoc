Ansible роль для миграции PostgreSQL

Ниже представлена Ansible роль для выполнения миграции базы данных PostgreSQL с одного сервера на другой.

Структура роли

```
roles/postgres_migration/
├── tasks/
│   ├── main.yml
│   ├── create_dump.yml
│   ├── copy_dump.yml
│   └── restore_dump.yml
├── defaults/
│   └── main.yml
└── templates/
    └── pgpass.j2
```

Файлы роли

defaults/main.yml

```yaml
---
# Исходный сервер PostgreSQL
source_postgres_host: "source-db.example.com"
source_postgres_port: 5432
source_postgres_user: "postgres"
source_postgres_password: "password"
source_postgres_db: "my_database"

# Целевой сервер PostgreSQL
target_postgres_host: "target-db.example.com"
target_postgres_port: 5432
target_postgres_user: "postgres"
target_postgres_password: "password"

# Настройки дампа
dump_dir: "/tmp/postgres_dumps"
dump_filename: "{{ source_postgres_db }}_dump_{{ ansible_date_time.iso8601 }}.sql"
dump_path: "{{ dump_dir }}/{{ dump_filename }}"
pg_dump_options: "--format=custom --blobs --verbose"
pg_restore_options: "--clean --create --verbose"

# Учетные данные для .pgpass
pgpass_path: "~/.pgpass"
```

tasks/create_dump.yml

```yaml
---
- name: Ensure dump directory exists on source host
  file:
    path: "{{ dump_dir }}"
    state: directory
    mode: '0755'

- name: Create .pgpass file on source host
  template:
    src: "pgpass.j2"
    dest: "{{ pgpass_path }}"
    mode: '0600'
  when: source_postgres_password != ""

- name: Create PostgreSQL dump
  command: >
    pg_dump {{ pg_dump_options }}
    --host={{ source_postgres_host }}
    --port={{ source_postgres_port }}
    --username={{ source_postgres_user }}
    --dbname={{ source_postgres_db }}
    --file={{ dump_path }}
  environment:
    PGPASSFILE: "{{ pgpass_path }}"
  register: dump_result
  changed_when: dump_result.rc == 0
  failed_when: dump_result.rc != 0

- name: Verify dump file was created
  stat:
    path: "{{ dump_path }}"
  register: dump_file

- name: Fail if dump file is empty
  fail:
    msg: "Dump file {{ dump_path }} is empty or not created"
  when: dump_file.stat.size == 0
```

tasks/copy_dump.yml

```yaml
---
- name: Ensure dump directory exists on target host
  file:
    path: "{{ dump_dir }}"
    state: directory
    mode: '0755'

- name: Copy dump file to target host
  synchronize:
    src: "{{ dump_path }}"
    dest: "{{ dump_path }}"
    mode: pull
```

tasks/restore_dump.yml

```yaml
---
- name: Create .pgpass file on target host
  template:
    src: "pgpass.j2"
    dest: "{{ pgpass_path }}"
    mode: '0600'
  when: target_postgres_password != ""

- name: Restore PostgreSQL dump
  command: >
    pg_restore {{ pg_restore_options }}
    --host={{ target_postgres_host }}
    --port={{ target_postgres_port }}
    --username={{ target_postgres_user }}
    --dbname={{ source_postgres_db }}
    "{{ dump_path }}"
  environment:
    PGPASSFILE: "{{ pgpass_path }}"
  register: restore_result
  changed_when: restore_result.rc == 0
  failed_when: restore_result.rc != 0
```

tasks/main.yml

```yaml
---
- name: Include create dump tasks
  include_tasks: create_dump.yml
  when: "'source' in ansible_play_hosts or inventory_hostname in groups['source']"

- name: Include copy dump tasks
  include_tasks: copy_dump.yml
  when: "'target' in ansible_play_hosts or inventory_hostname in groups['target']"

- name: Include restore dump tasks
  include_tasks: restore_dump.yml
  when: "'target' in ansible_play_hosts or inventory_hostname in groups['target']"
```

templates/pgpass.j2

```
{{ source_postgres_host }}:{{ source_postgres_port }}:*:{{ source_postgres_user }}:{{ source_postgres_password }}
{{ target_postgres_host }}:{{ target_postgres_port }}:*:{{ target_postgres_user }}:{{ target_postgres_password }}
```

Пример playbook

```yaml
---
- name: Migrate PostgreSQL database
  hosts: source,target
  gather_facts: yes
  become: yes

  roles:
    - postgres_migration
```

Использование

1. Создайте inventory файл с группами source и target:

```ini
[source]
source-db.example.com

[target]
target-db.example.com
```

1. Запустите playbook:

```bash
ansible-playbook -i inventory.ini migrate_postgres.yml
```

Примечания

1. Убедитесь, что на обоих серверах установлены клиентские утилиты PostgreSQL (postgresql-client или аналогичные).
2. Для больших баз данных рассмотрите возможность использования pg_dump с параллельным режимом (-j N).
3. Для инкрементальных миграций можно использовать логическую репликацию PostgreSQL.
4. Перед выполнением в production рекомендуется протестировать процесс на тестовой среде.