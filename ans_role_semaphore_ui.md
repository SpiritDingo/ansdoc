# Роль Ansible для установки и настройки Semaphore UI Enterprise

Вот полная роль Ansible для установки и настройки Semaphore UI Enterprise с использованием Docker Compose v2.

## Структура роли

```
roles/semaphore_ui/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── docker-compose.yml.j2
└── vars
    └── main.yml
```

## 1. defaults/main.yml

```yaml
---
# Настройки Semaphore UI Enterprise
semaphore_ui_version: "latest"
semaphore_ui_port: 3000
semaphore_ui_data_dir: "/opt/semaphore_ui/data"

# Настройки базы данных (PostgreSQL)
semaphore_db_host: "postgres"
semaphore_db_port: 5432
semaphore_db_name: "semaphore"
semaphore_db_user: "semaphore"
semaphore_db_password: "change_this_strong_password"

# Настройки администратора
semaphore_admin_email: "admin@example.com"
semaphore_admin_password: "change_this_admin_password"
semaphore_admin_name: "Admin"

# Настройки Docker
docker_compose_version: "2"
```

## 2. vars/main.yml

```yaml
---
# Docker образ Semaphore UI Enterprise
semaphore_ui_image: "semaphoreui/semaphore-ee:{{ semaphore_ui_version }}"
postgres_image: "postgres:15-alpine"

# Имя сервиса
semaphore_service_name: "semaphore_ui"
```

## 3. tasks/main.yml

```yaml
---
- name: Ensure required directories exist
  file:
    path: "{{ semaphore_ui_data_dir }}"
    state: directory
    mode: '0755'

- name: Ensure PostgreSQL data directory exists
  file:
    path: "{{ semaphore_ui_data_dir }}/postgres"
    state: directory
    mode: '0755'

- name: Deploy Docker Compose file for Semaphore UI
  template:
    src: "docker-compose.yml.j2"
    dest: "{{ semaphore_ui_data_dir }}/docker-compose.yml"
    mode: '0644'

- name: Pull Docker images
  community.docker.docker_compose:
    project_src: "{{ semaphore_ui_data_dir }}"
    pull: yes

- name: Start Semaphore UI services
  community.docker.docker_compose:
    project_src: "{{ semaphore_ui_data_dir }}"
    state: present
    restart: yes

- name: Wait for Semaphore UI to be ready
  uri:
    url: "http://localhost:{{ semaphore_ui_port }}/api/auth/login"
    method: GET
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 10
  delay: 10

- name: Display Semaphore UI access information
  debug:
    msg:
      - "Semaphore UI Enterprise has been successfully deployed!"
      - "Access URL: http://{{ ansible_host }}:{{ semaphore_ui_port }}"
      - "Admin email: {{ semaphore_admin_email }}"
      - "Admin password: {{ semaphore_admin_password }}"
```

## 4. templates/docker-compose.yml.j2

```yaml
version: '{{ docker_compose_version }}'

services:
  postgres:
    image: {{ postgres_image }}
    container_name: semaphore_postgres
    environment:
      POSTGRES_USER: {{ semaphore_db_user }}
      POSTGRES_PASSWORD: {{ semaphore_db_password }}
      POSTGRES_DB: {{ semaphore_db_name }}
    volumes:
      - {{ semaphore_ui_data_dir }}/postgres:/var/lib/postgresql/data
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{ semaphore_db_user }} -d {{ semaphore_db_name }}"]
      interval: 5s
      timeout: 5s
      retries: 5

  semaphore:
    image: {{ semaphore_ui_image }}
    container_name: {{ semaphore_service_name }}
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "{{ semaphore_ui_port }}:3000"
    environment:
      SEMAPHORE_DB_HOST: {{ semaphore_db_host }}
      SEMAPHORE_DB_PORT: {{ semaphore_db_port }}
      SEMAPHORE_DB_NAME: {{ semaphore_db_name }}
      SEMAPHORE_DB_USER: {{ semaphore_db_user }}
      SEMAPHORE_DB_PASS: {{ semaphore_db_password }}
      SEMAPHORE_ADMIN: {{ semaphore_admin_name }}
      SEMAPHORE_ADMIN_EMAIL: {{ semaphore_admin_email }}
      SEMAPHORE_ADMIN_PASSWORD: {{ semaphore_admin_password }}
      SEMAPHORE_ACCESS_KEY_ENCRYPTION: "{{ 64 | random | to_uuid }}"
    volumes:
      - {{ semaphore_ui_data_dir }}:/etc/semaphore
    restart: unless-stopped
```

## Использование роли

1. Создайте playbook (например, `install_semaphore.yml`):

```yaml
---
- hosts: semaphore_servers
  become: yes
  roles:
    - role: semaphore_ui
      vars:
        semaphore_ui_version: "2.8.0"  # Укажите нужную версию
        semaphore_ui_port: 8080
        semaphore_admin_password: "your_strong_password_here"
        semaphore_db_password: "your_db_password_here"
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini install_semaphore.yml
```

## Дополнительные настройки

Вы можете добавить следующие параметры в `defaults/main.yml` для дополнительной конфигурации:

```yaml
# Настройки LDAP (если требуется)
semaphore_ldap_enabled: false
semaphore_ldap_host: ""
semaphore_ldap_port: 389
semaphore_ldap_dn: ""
semaphore_ldap_bind_dn: ""
semaphore_ldap_bind_password: ""
semaphore_ldap_search_filter: ""
semaphore_ldap_mapping: ""

# Настройки SMTP (для уведомлений)
semaphore_smtp_host: ""
semaphore_smtp_port: 587
semaphore_smtp_user: ""
semaphore_smtp_password: ""
semaphore_smtp_from: ""
```

Затем обновите шаблон `docker-compose.yml.j2`, добавив соответствующие переменные окружения.

Эта роль обеспечит установку и настройку Semaphore UI Enterprise с использованием Docker Compose v2, включая настройку базы данных PostgreSQL и административного пользователя.