# Роль Ansible для переноса базы данных PostgreSQL на новый сервер

Ниже представлена Ansible роль для переноса базы данных PostgreSQL с одного сервера на другой.

## Структура роли

```
roles/postgresql_migration/
├── defaults
│   └── main.yml       # Переменные по умолчанию
├── tasks
│   └── main.yml       # Основные задачи
├── templates          # Шаблоны конфигураций
└── vars
    └── main.yml       # Основные переменные
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Исходный сервер PostgreSQL
source_postgres_host: "old-server.example.com"
source_postgres_port: 5432
source_postgres_user: "postgres"
source_postgres_password: ""

# Целевой сервер PostgreSQL
target_postgres_host: "new-server.example.com"
target_postgres_port: 5432
target_postgres_user: "postgres"
target_postgres_password: ""

# Настройки базы данных
db_name: "my_database"
db_owner: "db_user"
dump_file: "/tmp/{{ db_name }}_dump.sql"
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей на исходном сервере
  become: yes
  apt:
    name: postgresql-client
    state: present
  when: ansible_os_family == 'Debian'
  delegate_to: "{{ source_postgres_host }}"

- name: Создание дампа базы данных
  become: yes
  become_user: postgres
  command: >
    pg_dump -U {{ source_postgres_user }} -h {{ source_postgres_host }} 
    -p {{ source_postgres_port }} -F c -f {{ dump_file }} {{ db_name }}
  args:
    creates: "{{ dump_file }}"
  delegate_to: "{{ source_postgres_host }}"
  register: dump_result
  changed_when: "'already exists' not in dump_result.stderr"

- name: Копирование дампа на целевой сервер
  fetch:
    src: "{{ dump_file }}"
    dest: "{{ dump_file }}"
    flat: yes
  delegate_to: "{{ source_postgres_host }}"

- name: Установка PostgreSQL на целевом сервере
  become: yes
  apt:
    name: postgresql
    state: present
  when: ansible_os_family == 'Debian'

- name: Создание пользователя базы данных на целевом сервере
  become: yes
  become_user: postgres
  postgresql_user:
    name: "{{ db_owner }}"
    password: "{{ db_owner_password | default(omit) }}"
    state: present
  when: db_owner is defined

- name: Восстановление базы данных из дампа
  become: yes
  become_user: postgres
  command: >
    pg_restore -U {{ target_postgres_user }} -h {{ target_postgres_host }}
    -p {{ target_postgres_port }} -C -d postgres {{ dump_file }}
  args:
    creates: "/var/lib/postgresql/{{ target_postgres_version }}/main/base/{{ db_oid }}"
  register: restore_result
  changed_when: restore_result.rc == 0
```

## Пример playbook для использования роли

```yaml
---
- name: Миграция PostgreSQL базы данных
  hosts: localhost
  gather_facts: yes
  vars:
    source_postgres_host: "db-old.example.com"
    target_postgres_host: "db-new.example.com"
    db_name: "production_db"
    db_owner: "app_user"
    db_owner_password: "securepassword123"
  
  roles:
    - postgresql_migration
```

## Дополнительные рекомендации

1. **Безопасность**: Используйте Ansible Vault для хранения паролей:
   ```bash
   ansible-vault encrypt_string 'securepassword123' --name 'db_owner_password'
   ```

2. **Большие базы данных**: Для больших БД рассмотрите использование:
   - `pg_dump -Fd` для параллельного дампа
   - Сжатие при передаче
   - Прямую передачу через pipe без временных файлов

3. **Проверка**: Добавьте задачи для проверки целостности данных после миграции.

4. **Downtime**: Запланируйте downtime приложения или используйте репликацию для минимального простоя.

5. **Версии PostgreSQL**: Убедитесь, что версии PostgreSQL совместимы между исходным и целевым серверами.