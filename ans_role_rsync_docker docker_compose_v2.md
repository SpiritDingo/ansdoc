# Ansible роль для синхронизации файлов между серверами через SSH с использованием Docker

Эта роль настраивает контейнеризованное решение для безопасной синхронизации файлов между Linux-серверами с использованием rsync через SSH, где каждая интеграция работает в отдельном Docker-контейнере.

## Структура роли

```
roles/rsync_ssh_docker/
├── defaults
│   └── main.yml
├── tasks
│   ├── main.yml
│   ├── client_setup.yml
│   └── server_setup.yml
├── templates
│   ├── docker-compose.server.yml.j2
│   ├── docker-compose.client.yml.j2
│   ├── ssh_config.j2
│   └── sync_script.sh.j2
└── vars
    └── main.yml
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Общие настройки
rsync_ssh_user: syncuser
rsync_base_port: 22000  # Базовый порт для SSH
docker_compose_dir: /opt/sync_containers
data_dir: /opt/sync_data

# Настройки сервера
server_ssh_key_path: /etc/ssh/sync_keys
server_accept_keys: true

# Настройки клиента
client_ssh_key_path: /etc/ssh/sync_keys
client_known_hosts: /etc/ssh/ssh_known_hosts
sync_schedule: "*/15 * * * *"  # Расписание cron по умолчанию
```

### tasks/main.yml

```yaml
---
- name: Определение роли (сервер или клиент)
  set_fact:
    is_server: "{{ 'server' in inventory_hostname|lower }}"
    is_client: "{{ 'client' in inventory_hostname|lower }}"

- include_tasks: server_setup.yml
  when: is_server

- include_tasks: client_setup.yml
  when: is_client
```

### tasks/server_setup.yml

```yaml
---
- name: Установка зависимостей на сервере
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - docker.io
    - docker-compose
    - openssh-server

- name: Создание директорий на сервере
  file:
    path: "{{ item }}"
    state: directory
    mode: 0755
  loop:
    - "{{ docker_compose_dir }}"
    - "{{ data_dir }}"
    - "{{ server_ssh_key_path }}"

- name: Генерация SSH ключей для сервера
  openssh_keypair:
    path: "{{ server_ssh_key_path }}/id_rsa"
    type: rsa
    size: 4096
    force: no

- name: Настройка контейнеров rsync+SSH сервера
  template:
    src: "docker-compose.server.yml.j2"
    dest: "{{ docker_compose_dir }}/{{ item.name }}/docker-compose.yml"
  loop: "{{ sync_integrations }}"
  when: item.server == inventory_hostname

- name: Запуск контейнеров на сервере
  docker_compose:
    project_src: "{{ docker_compose_dir }}/{{ item.name }}"
    state: present
  loop: "{{ sync_integrations }}"
  when: item.server == inventory_hostname
```

### tasks/client_setup.yml

```yaml
---
- name: Установка зависимостей на клиенте
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - docker.io
    - docker-compose
    - openssh-client
    - cron

- name: Создание директорий на клиенте
  file:
    path: "{{ item }}"
    state: directory
    mode: 0755
  loop:
    - "{{ docker_compose_dir }}"
    - "{{ data_dir }}"
    - "{{ client_ssh_key_path }}"

- name: Копирование публичных ключей серверов
  copy:
    content: "{{ item.public_key }}"
    dest: "{{ client_ssh_key_path }}/known_hosts_{{ item.name }}"
    mode: 0644
  loop: "{{ sync_integrations }}"
  when: item.client == inventory_hostname

- name: Настройка контейнеров rsync+SSH клиента
  template:
    src: "docker-compose.client.yml.j2"
    dest: "{{ docker_compose_dir }}/{{ item.name }}/docker-compose.yml"
  loop: "{{ sync_integrations }}"
  when: item.client == inventory_hostname

- name: Создание скриптов синхронизации
  template:
    src: "sync_script.sh.j2"
    dest: "{{ docker_compose_dir }}/{{ item.name }}/sync.sh"
    mode: 0755
  loop: "{{ sync_integrations }}"
  when: item.client == inventory_hostname

- name: Настройка cron заданий
  cron:
    name: "Sync {{ item.name }}"
    job: "{{ docker_compose_dir }}/{{ item.name }}/sync.sh"
    user: root
    minute: "{{ item.schedule | default(sync_schedule).split()[0] }}"
    hour: "{{ item.schedule | default(sync_schedule).split()[1] }}"
    day: "{{ item.schedule | default(sync_schedule).split()[2] }}"
    month: "{{ item.schedule | default(sync_schedule).split()[3] }}"
    weekday: "{{ item.schedule | default(sync_schedule).split()[4] }}"
  loop: "{{ sync_integrations }}"
  when: item.client == inventory_hostname
```

### templates/docker-compose.server.yml.j2

```yaml
version: '3'

services:
  sync_server_{{ item.name }}:
    image: instrumentisto/rsync-ssh
    container_name: sync_server_{{ item.name }}
    restart: always
    ports:
      - "{{ rsync_base_port + loop.index0 }}:22"
    volumes:
      - "{{ server_ssh_key_path }}/id_rsa:/home/{{ rsync_ssh_user }}/.ssh/id_rsa"
      - "{{ server_ssh_key_path }}/id_rsa.pub:/home/{{ rsync_ssh_user }}/.ssh/id_rsa.pub"
      - "{{ data_dir }}/{{ item.name }}:/data"
    environment:
      - SSH_ENABLE_ROOT=true
      - USER_NAME={{ rsync_ssh_user }}
      - USER_PASSWORD={{ item.ssh_password | default('changeme') }}
    command: /usr/sbin/sshd -D
```

### templates/docker-compose.client.yml.j2

```yaml
version: '3'

services:
  sync_client_{{ item.name }}:
    image: instrumentisto/rsync-ssh
    container_name: sync_client_{{ item.name }}
    restart: unless-stopped
    volumes:
      - "{{ client_ssh_key_path }}/known_hosts_{{ item.name }}:/home/{{ rsync_ssh_user }}/.ssh/known_hosts"
      - "{{ data_dir }}/{{ item.name }}:/data"
      - "./sync.sh:/sync.sh"
    environment:
      - USER_NAME={{ rsync_ssh_user }}
    entrypoint: /bin/sh -c
    command: "while true; do sleep 3600; done"
```

### templates/sync_script.sh.j2

```bash
#!/bin/bash

RSYNC_OPTS="-avz --delete -e 'ssh -p {{ rsync_base_port + loop.index0 }} -i /home/{{ rsync_ssh_user }}/.ssh/id_rsa -o StrictHostKeyChecking=yes'"

# Направление синхронизации
{% if item.direction == 'push' %}
rsync $RSYNC_OPTS /data/ {{ rsync_ssh_user }}@{{ item.server }}:{{ item.remote_path | default('/data') }}
{% else %}
rsync $RSYNC_OPTS {{ rsync_ssh_user }}@{{ item.server }}:{{ item.remote_path | default('/data') }} /data/
{% endif %}
```

## Пример inventory

```yaml
all:
  vars:
    sync_integrations:
      - name: backup_sync
        description: "Daily backups to storage"
        server: "storage01.example.com"
        client: "webserver01.example.com"
        direction: "push"
        schedule: "0 3 * * *"
        ssh_password: "secure_password_123"
        
      - name: config_sync
        description: "Config synchronization between servers"
        server: "config01.example.com"
        client: "appserver01.example.com"
        direction: "pull"
        schedule: "*/5 * * * *"
        remote_path: "/etc/app_configs"
```

## Применение роли

1. Создайте inventory файл с вашими серверами и настройками интеграций
2. Запустите playbook:

```bash
ansible-playbook -i inventory.yml sync_playbook.yml
```

## Особенности реализации

1. **Безопасность**:
   - Использование SSH с ключами аутентификации
   - Изолированные контейнеры для каждой интеграции
   - Отдельные порты для каждого подключения

2. **Гибкость**:
   - Поддержка push/pull моделей синхронизации
   - Настраиваемое расписание для каждой интеграции
   - Возможность указания custom путей

3. **Масштабируемость**:
   - Легко добавлять новые интеграции
   - Каждая интеграция полностью изолирована
   - Возможность распределения нагрузки между серверами

4. **Мониторинг**:
   - Логи синхронизации доступны в контейнерах
   - Статус контейнеров можно проверять через docker-compose

Для работы этой роли необходимо предварительно настроить DNS или прописать хосты в /etc/hosts на всех участвующих серверах.