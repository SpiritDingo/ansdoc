# Ansible роль для установки NetBox в Docker Compose

Ниже представлена структура и содержимое Ansible роли для развертывания NetBox с использованием Docker Compose.

## Структура роли

```
netbox-docker/
├── defaults/
│   └── main.yml          # Значения переменных по умолчанию
├── files/
│   ├── docker-compose.yml.j2  # Шаблон docker-compose файла
│   └── env.netbox.j2     # Шаблон файла с переменными окружения
├── tasks/
│   └── main.yml          # Основные задачи установки
└── vars/
    └── main.yml          # Основные переменные роли
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Версия NetBox
netbox_version: "v3.6.2"

# Директория для установки
netbox_install_dir: "/opt/netbox-docker"

# Настройки базы данных PostgreSQL
netbox_db_name: "netbox"
netbox_db_user: "netbox"
netbox_db_password: "changeme"

# Настройки Redis
netbox_redis_password: ""

# Настройки администратора
netbox_admin_name: "Admin"
netbox_admin_email: "admin@example.com"
netbox_admin_password: "admin"

# Настройки сервера
netbox_server_name: "netbox.example.com"
netbox_secret_key: "generate-me-with-ansible-vault"

# Настройки Docker
docker_compose_version: "1.29.2"
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  apt:
    name:
      - docker.io
      - docker-compose
      - git
    state: present
    update_cache: yes

- name: Добавление пользователя в группу docker
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Создание директории для NetBox
  file:
    path: "{{ netbox_install_dir }}"
    state: directory
    mode: '0755'

- name: Копирование docker-compose.yml
  template:
    src: "docker-compose.yml.j2"
    dest: "{{ netbox_install_dir }}/docker-compose.yml"
    mode: '0644'

- name: Копирование файла переменных окружения
  template:
    src: "env.netbox.j2"
    dest: "{{ netbox_install_dir }}/env/netbox.env"
    mode: '0640'

- name: Запуск контейнеров NetBox
  community.docker.docker_compose:
    project_src: "{{ netbox_install_dir }}"
    build: yes
    pull: yes
    recreate: always
    restart: yes

- name: Ожидание готовности NetBox
  uri:
    url: "http://localhost:8080"
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 10
  delay: 10

- name: Создание суперпользователя (если требуется)
  command: >
    docker-compose -f {{ netbox_install_dir }}/docker-compose.yml
    exec -T netbox /opt/netbox/netbox/manage.py createsuperuser
    --noinput --username admin --email {{ netbox_admin_email }}
  when: netbox_create_superuser
```

### files/docker-compose.yml.j2

```yaml
version: '3.4'
services:
  postgres:
    image: postgres:15-alpine
    env_file: env/postgres.env
    volumes:
      - postgres-data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass {{ netbox_redis_password }}
    volumes:
      - redis-data:/data
    restart: unless-stopped

  redis-cache:
    image: redis:7-alpine
    command: redis-server --requirepass {{ netbox_redis_password }}
    volumes:
      - redis-cache-data:/data
    restart: unless-stopped

  netbox:
    image: netboxcommunity/netbox:{{ netbox_version }}
    depends_on:
      - postgres
      - redis
      - redis-cache
    env_file: env/netbox.env
    volumes:
      - netbox-media-files:/opt/netbox/netbox/media
      - netbox-static-files:/opt/netbox/netbox/static
    ports:
      - "8080:8080"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/"]
      interval: 1m
      timeout: 5s
      retries: 5

  netbox-worker:
    image: netboxcommunity/netbox:{{ netbox_version }}
    depends_on:
      - postgres
      - redis
    env_file: env/netbox.env
    command: /opt/netbox/venv/bin/python /opt/netbox/netbox/manage.py rqworker
    volumes:
      - netbox-media-files:/opt/netbox/netbox/media
    restart: unless-stopped

  netbox-housekeeping:
    image: netboxcommunity/netbox:{{ netbox_version }}
    depends_on:
      - postgres
      - redis
    env_file: env/netbox.env
    command: /opt/netbox/housekeeping.sh
    volumes:
      - netbox-media-files:/opt/netbox/netbox/media
    restart: unless-stopped

volumes:
  postgres-data:
  redis-data:
  redis-cache-data:
  netbox-media-files:
  netbox-static-files:
```

### files/env.netbox.j2

```ini
# NetBox configuration
ALLOWED_HOSTS={{ netbox_server_name }}
SECRET_KEY={{ netbox_secret_key }}

# PostgreSQL configuration
DB_NAME={{ netbox_db_name }}
DB_USER={{ netbox_db_user }}
DB_PASSWORD={{ netbox_db_password }}
DB_HOST=postgres
DB_PORT=5432

# Redis configuration
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD={{ netbox_redis_password }}
REDIS_DB=0
REDIS_CACHE_HOST=redis-cache
REDIS_CACHE_PORT=6379
REDIS_CACHE_PASSWORD={{ netbox_redis_password }}
REDIS_CACHE_DB=0

# Email configuration
EMAIL_SERVER=localhost
EMAIL_PORT=25
EMAIL_USERNAME=
EMAIL_PASSWORD=
EMAIL_TIMEOUT=10
EMAIL_FROM=netbox@{{ netbox_server_name }}

# Admin user configuration (for initial setup)
SUPERUSER_NAME={{ netbox_admin_name }}
SUPERUSER_EMAIL={{ netbox_admin_email }}
SUPERUSER_PASSWORD={{ netbox_admin_password }}
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
---
- hosts: netbox_server
  become: yes
  roles:
    - netbox-docker
```

2. Создайте файл с переменными для переопределения значений по умолчанию:

```yaml
---
netbox_db_password: "secure_db_password"
netbox_redis_password: "secure_redis_password"
netbox_secret_key: "very_secret_key_generated_with_openssl"
netbox_server_name: "netbox.mycompany.com"
netbox_admin_email: "admin@mycompany.com"
```

3. Запустите playbook:

```bash
ansible-playbook -i inventory.ini netbox_docker_playbook.yml --extra-vars "@vars.yml"
```

## Дополнительные рекомендации

1. Для генерации секретного ключа можно использовать команду:
```bash
openssl rand -base64 64 | tr -d '\n'
```

2. Для защиты чувствительных данных используйте Ansible Vault:
```bash
ansible-vault encrypt vars.yml
```

3. После установки NetBox будет доступен на порту 8080. Для настройки HTTPS и доменного имени можно добавить Nginx в качестве обратного прокси.

4. Для резервного копирования данных Docker-контейнеров NetBox нужно сохранить:
- Тома Docker (postgres-data, redis-data и т.д.)
- Файлы конфигурации (docker-compose.yml и env/*.env)