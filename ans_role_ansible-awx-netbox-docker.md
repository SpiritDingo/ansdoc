# Ansible, AWX и Netbox в Docker-compose: Установка и интеграция

В этом руководстве я предоставлю роль Ansible для развертывания Ansible, AWX (веб-интерфейс для Ansible) и Netbox (IPAM/DCIM система) с их интеграцией через Docker-compose.

## Структура роли

Создадим структуру роли:

```
ansible-awx-netbox-docker/
├── defaults/
│   └── main.yml
├── files/
│   ├── docker-compose.yml
│   ├── awx/
│   │   ├── projects/
│   │   └── credentials/
│   └── netbox/
│       └── configuration.py
├── tasks/
│   └── main.yml
├── templates/
│   └── netbox.env.j2
└── vars/
    └── main.yml
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Настройки Docker
docker_compose_version: "1.29.2"

# Настройки Netbox
netbox_version: "v3.6.1"
netbox_db_password: "changeme"
netbox_secret_key: "changeme"
netbox_admin_user: "admin"
netbox_admin_email: "admin@example.com"
netbox_admin_password: "admin"

# Настройки AWX
awx_version: "21.12.0"
awx_admin_user: "admin"
awx_admin_password: "changeme"
awx_secret_key: "changeme"
awx_postgres_password: "changeme"
awx_rabbitmq_password: "changeme"
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
  loop:
    - docker.io
    - docker-compose
    - python3-pip
    - git

- name: Установка docker-compose через pip (если нужна новая версия)
  pip:
    name: docker-compose
    version: "{{ docker_compose_version }}"
    state: present

- name: Добавление пользователя в группу docker
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Создание директорий для конфигурации
  file:
    path: "{{ item }}"
    state: directory
    mode: 0755
  loop:
    - /opt/awx/projects
    - /opt/awx/credentials
    - /opt/netbox

- name: Копирование docker-compose файла
  copy:
    src: files/docker-compose.yml
    dest: /opt/docker-compose.yml
    mode: 0644

- name: Копирование конфигурации Netbox
  copy:
    src: files/netbox/configuration.py
    dest: /opt/netbox/configuration.py
    mode: 0644

- name: Генерация файла окружения Netbox
  template:
    src: templates/netbox.env.j2
    dest: /opt/netbox/netbox.env
    mode: 0644

- name: Запуск сервисов через docker-compose
  command: docker-compose -f /opt/docker-compose.yml up -d
  args:
    chdir: /opt

- name: Ожидание готовности Netbox
  uri:
    url: "http://localhost:8000"
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 10
  delay: 10

- name: Создание суперпользователя Netbox
  command: >
    docker exec -it netbox python /opt/netbox/netbox/manage.py createsuperuser
    --no-input
    --username "{{ netbox_admin_user }}"
    --email "{{ netbox_admin_email }}"
  when: netbox_admin_password != ""

- name: Установка плагина Netbox для AWX
  pip:
    name: awx-netbox-inventory
    state: present
    executable: pip3

- name: Настройка интеграции AWX с Netbox
  uri:
    url: "http://localhost:8052/api/v2/inventory_sources/"
    method: POST
    user: "{{ awx_admin_user }}"
    password: "{{ awx_admin_password }}"
    body_format: json
    body:
      name: "Netbox Inventory"
      source: "scm"
      source_path: "https://github.com/ansible-collections/netbox.git"
      source_project: "Netbox Collection"
      inventory: "Netbox"
      update_on_launch: true
    status_code: 201
    force_basic_auth: yes
```

### files/docker-compose.yml

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: netbox
      POSTGRES_USER: netbox
      POSTGRES_PASSWORD: ${NETBOX_DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:6.2
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - redis_data:/data
    restart: unless-stopped

  netbox:
    image: netboxcommunity/netbox:${NETBOX_VERSION}
    depends_on:
      - postgres
      - redis
    environment:
      - DB_NAME=netbox
      - DB_USER=netbox
      - DB_PASSWORD=${NETBOX_DB_PASSWORD}
      - DB_HOST=postgres
      - DB_PORT=5432
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_DB=0
      - SECRET_KEY=${NETBOX_SECRET_KEY}
      - SUPERUSER_NAME=${NETBOX_ADMIN_USER}
      - SUPERUSER_EMAIL=${NETBOX_ADMIN_EMAIL}
      - SUPERUSER_PASSWORD=${NETBOX_ADMIN_PASSWORD}
    volumes:
      - ./netbox/configuration.py:/etc/netbox/config/configuration.py:ro,z
    ports:
      - "8000:8080"
    restart: unless-stopped

  awx-postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: awx
      POSTGRES_PASSWORD: ${AWX_POSTGRES_PASSWORD}
      POSTGRES_DB: awx
    volumes:
      - awx_postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  awx-rabbitmq:
    image: rabbitmq:3.8-management
    environment:
      RABBITMQ_DEFAULT_USER: awx
      RABBITMQ_DEFAULT_PASS: ${AWX_RABBITMQ_PASSWORD}
    volumes:
      - awx_rabbitmq_data:/var/lib/rabbitmq
    restart: unless-stopped

  awx-memcached:
    image: memcached:alpine
    entrypoint: memcached -m 64
    restart: unless-stopped

  awx:
    image: ansible/awx:${AWX_VERSION}
    depends_on:
      - awx-postgres
      - awx-rabbitmq
      - awx-memcached
    environment:
      AWX_ADMIN_USER: ${AWX_ADMIN_USER}
      AWX_ADMIN_PASSWORD: ${AWX_ADMIN_PASSWORD}
      AWX_SECRET_KEY: ${AWX_SECRET_KEY}
      AWX_POSTGRES_HOST: awx-postgres
      AWX_POSTGRES_PORT: 5432
      AWX_POSTGRES_USER: awx
      AWX_POSTGRES_PASSWORD: ${AWX_POSTGRES_PASSWORD}
      AWX_POSTGRES_DB: awx
      AWX_RABBITMQ_HOST: awx-rabbitmq
      AWX_RABBITMQ_PORT: 5672
      AWX_RABBITMQ_USER: awx
      AWX_RABBITMQ_PASSWORD: ${AWX_RABBITMQ_PASSWORD}
      AWX_RABBITMQ_VHOST: awx
      AWX_MEMCACHED_HOST: awx-memcached
      AWX_MEMCACHED_PORT: 11211
    volumes:
      - ./awx/projects:/var/lib/awx/projects:z
      - ./awx/credentials:/var/lib/awx/credentials:z
    ports:
      - "8052:8052"
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  awx_postgres_data:
  awx_rabbitmq_data:
```

### templates/netbox.env.j2

```jinja2
NETBOX_VERSION={{ netbox_version }}
NETBOX_DB_PASSWORD={{ netbox_db_password }}
NETBOX_SECRET_KEY={{ netbox_secret_key }}
NETBOX_ADMIN_USER={{ netbox_admin_user }}
NETBOX_ADMIN_EMAIL={{ netbox_admin_email }}
NETBOX_ADMIN_PASSWORD={{ netbox_admin_password }}

AWX_VERSION={{ awx_version }}
AWX_ADMIN_USER={{ awx_admin_user }}
AWX_ADMIN_PASSWORD={{ awx_admin_password }}
AWX_SECRET_KEY={{ awx_secret_key }}
AWX_POSTGRES_PASSWORD={{ awx_postgres_password }}
AWX_RABBITMQ_PASSWORD={{ awx_rabbitmq_password }}
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
---
- name: Установка Ansible, AWX и Netbox с интеграцией
  hosts: target_server
  become: yes
  roles:
    - ansible-awx-netbox-docker
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini deploy.yml
```

## Интеграция AWX и Netbox

После развертывания вы можете:
1. Использовать Netbox как источник инвентаризации в AWX
2. Создавать динамические инвентари на основе данных Netbox
3. Автоматизировать управление устройствами в Netbox через AWX

## Дальнейшая настройка

После установки рекомендуется:
1. Настроить TLS/SSL для AWX и Netbox
2. Настроить резервное копирование баз данных
3. Настроить аутентификацию (LDAP/OAuth)
4. Настроить регулярные синхронизации инвентаризации из Netbox в AWX

Эта роль предоставляет базовую установку и интеграцию, которую можно расширять в соответствии с вашими потребностями.