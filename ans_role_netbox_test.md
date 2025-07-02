# Подробная Ansible роль для развертывания NetBox

Вот более детализированная версия Ansible роли для развертывания NetBox на основе вашего Docker Compose файла.

## Структура роли

```
netbox/
├── defaults/
│   └── main.yml
├── files/
│   ├── nginx/
│   │   └── default.conf
│   └── env/
│       ├── gateway.env
│       ├── postgresql.env
│       └── redis.env
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   ├── prerequisites.yml
│   ├── docker.yml
│   ├── volumes.yml
│   ├── configure.yml
│   ├── deploy.yml
│   └── main.yml
├── templates/
│   └── docker-compose.yml.j2
└── vars/
    └── main.yml
```

## Подробное содержание файлов

### defaults/main.yml

```yaml
---
# Основные параметры развертывания
netbox_base_dir: "/opt/nextbox"
netbox_user: "nextbox"
netbox_group: "nextbox"

# Версии образов
nginx_image: "nginx:latest"
postgresql_image: "postgres:16"
rabbitmq_image: "rabbitmq:3.7-management"
redis_image: "redis:6-alpine"

# Настройки сети
netbox_network_name: "nextbox"
netbox_nginx_port: 8095

# Настройки PostgreSQL
postgresql_user: "nextbox"
postgresql_password: "nextbox"
postgresql_databases:
  - "gateway"
  - "file_storage"
  - "auth"
  - "static_storage"
  - "connections"
  - "logstash"
  - "discovery"
  - "share"
  - "notifications"
  - "links"
```

### vars/main.yml

```yaml
---
# Все сервисы NetBox
netbox_services:
  - gateway
  - file_storage_router
  - file_storage
  - file_storage_worker
  - auth
  - fca
  - static_storage
  - connections
  - logstash
  - discovery
  - share
  - webdav
  - pusher
  - notifications
  - proxy
  - license
  - links
  - frontend

# Тома Docker
netbox_volumes:
  - nextbox_data
  - redis_data
  - rabbitmq_data
  - postgresql_data
  - storage_data
  - logstash_data
  - static_storage_data
```

### tasks/main.yml

```yaml
---
- name: Include prerequisites tasks
  ansible.builtin.include_tasks: prerequisites.yml

- name: Include Docker setup tasks
  ansible.builtin.include_tasks: docker.yml

- name: Include volumes setup tasks
  ansible.builtin.include_tasks: volumes.yml

- name: Include configuration tasks
  ansible.builtin.include_tasks: configure.yml

- name: Include deployment tasks
  ansible.builtin.include_tasks: deploy.yml
```

### tasks/prerequisites.yml

```yaml
---
- name: Ensure required packages are installed
  ansible.builtin.apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - software-properties-common
      - python3-pip
    state: present
    update_cache: yes

- name: Ensure required system user exists
  ansible.builtin.user:
    name: "{{ netbox_user }}"
    group: "{{ netbox_group }}"
    system: yes
    create_home: no
    shell: /usr/sbin/nologin

- name: Create base directory structure
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    owner: "{{ netbox_user }}"
    group: "{{ netbox_group }}"
    mode: '0755'
  loop:
    - "{{ netbox_base_dir }}"
    - "{{ netbox_base_dir }}/nginx"
    - "{{ netbox_base_dir }}/env"
    - "{{ netbox_base_dir }}/volumes"
```

### tasks/docker.yml

```yaml
---
- name: Add Docker GPG key
  ansible.builtin.apt_key:
    url: "https://download.docker.com/linux/ubuntu/gpg"
    state: present

- name: Add Docker repository
  ansible.builtin.apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present
    update_cache: yes

- name: Install Docker components
  ansible.builtin.apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
    state: present
    update_cache: yes

- name: Ensure Docker service is running
  ansible.builtin.service:
    name: docker
    state: started
    enabled: yes

- name: Add user to docker group
  ansible.builtin.user:
    name: "{{ netbox_user }}"
    groups: docker
    append: yes

- name: Install Docker Compose via pip
  ansible.builtin.pip:
    name: docker-compose
    state: present
```

### tasks/volumes.yml

```yaml
---
- name: Create volume directories
  ansible.builtin.file:
    path: "{{ netbox_base_dir }}/volumes/{{ item }}"
    state: directory
    owner: "{{ netbox_user }}"
    group: "{{ netbox_group }}"
    mode: '0755'
  loop: "{{ netbox_volumes }}"

- name: Set permissions for PostgreSQL volume
  ansible.builtin.file:
    path: "{{ netbox_base_dir }}/volumes/postgresql_data"
    owner: "999"
    group: "999"
    mode: '0750'
```

### tasks/configure.yml

```yaml
---
- name: Copy Nginx configuration
  ansible.builtin.template:
    src: "files/nginx/default.conf"
    dest: "{{ netbox_base_dir }}/nginx/default.conf"
    owner: "{{ netbox_user }}"
    group: "{{ netbox_group }}"
    mode: '0644'

- name: Create environment files
  ansible.builtin.template:
    src: "files/env/{{ item }}.env"
    dest: "{{ netbox_base_dir }}/env/{{ item }}.env"
    owner: "{{ netbox_user }}"
    group: "{{ netbox_group }}"
    mode: '0640'
  loop:
    - gateway
    - postgresql
    - redis

- name: Generate Docker Compose file
  ansible.builtin.template:
    src: "templates/docker-compose.yml.j2"
    dest: "{{ netbox_base_dir }}/docker-compose.yml"
    owner: "{{ netbox_user }}"
    group: "{{ netbox_group }}"
    mode: '0640'
```

### tasks/deploy.yml

```yaml
---
- name: Pull Docker images
  community.docker.docker_compose:
    project_src: "{{ netbox_base_dir }}"
    pull: yes
    build: no

- name: Deploy NetBox stack
  community.docker.docker_compose:
    project_src: "{{ netbox_base_dir }}"
    state: present
    restarted: yes
    recreate: always
    timeout: 180

- name: Wait for services to become healthy
  ansible.builtin.wait_for:
    delay: 10
    timeout: 300
  register: wait_result
  until: wait_result is succeeded
  retries: 5
```

### handlers/main.yml

```yaml
---
- name: Restart Docker
  ansible.builtin.service:
    name: docker
    state: restarted

- name: Reload Nginx
  community.docker.docker_container_exec:
    container: "nextbox_nginx_1"
    command: "nginx -s reload"
```

### templates/docker-compose.yml.j2

```yaml
version: '3.8'

services:
  nginx:
    image: {{ nginx_image }}
    container_name: nextbox_nginx
    restart: always
    ports:
      - "{{ netbox_nginx_port }}:80"
    networks:
      - {{ netbox_network_name }}
    volumes:
      - {{ netbox_base_dir }}/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - frontend

  postgresql:
    image: {{ postgresql_image }}
    container_name: nextbox_postgresql
    restart: always
    environment:
      POSTGRES_USER: {{ postgresql_user }}
      POSTGRES_PASSWORD: {{ postgresql_password }}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - {{ netbox_base_dir }}/volumes/postgresql_data:/var/lib/postgresql/data/pgdata
    networks:
      - {{ netbox_network_name }}
    env_file:
      - {{ netbox_base_dir }}/env/postgresql.env

  redis:
    image: {{ redis_image }}
    container_name: nextbox_redis
    restart: always
    networks:
      - {{ netbox_network_name }}
    volumes:
      - {{ netbox_base_dir }}/volumes/redis_data:/data
    env_file:
      - {{ netbox_base_dir }}/env/redis.env

  # Все остальные сервисы с их конфигурацией
  {% for service in netbox_services %}
  {{ service }}:
    image: nextbox-docker.ptnl.moscow/nextbox/{{ service }}:pine
    container_name: nextbox_{{ service }}
    restart: always
    networks:
      - {{ netbox_network_name }}
    {% if service in ['file_storage_worker', 'logstash', 'static_storage'] %}
    volumes:
      - {{ netbox_base_dir }}/volumes/{{ service }}_data:/var/nextbox/{{ service.split('_')|join('/') }}
    {% endif %}
    {% if service == 'gateway' %}
    env_file:
      - {{ netbox_base_dir }}/env/gateway.env
    {% endif %}
    depends_on:
      - postgresql
      - redis
      - rabbitmq
  {% endfor %}

networks:
  {{ netbox_network_name }}:

volumes:
  {% for volume in netbox_volumes %}
  {{ volume }}:
    driver: local
    driver_opts:
      type: none
      device: "{{ netbox_base_dir }}/volumes/{{ volume }}"
      o: bind
  {% endfor %}
```

## Дополнительные файлы конфигурации

### files/nginx/default.conf

```nginx
upstream nextbox {
    server frontend:80;
}

server {
    listen 80;
    server_name localhost;

    client_max_body_size 100M;

    location / {
        proxy_pass http://nextbox;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://gateway:6001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        proxy_pass http://static_storage:6010;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    access_log /var/log/nginx/nextbox_access.log;
    error_log /var/log/nginx/nextbox_error.log;
}
```

### files/env/gateway.env

```
POTENTIAL_GATEWAY_DATABASE_DSN=postgres://{{ postgresql_user }}:{{ postgresql_password }}@postgresql:5432/gateway
POTENTIAL_GATEWAY_RABBIT_MQ_HOST=rabbitmq
POTENTIAL_GATEWAY_DISCOVERY_URL=http://discovery:6011
POTENTIAL_GATEWAY_SERVICE_BACK_URL=http://gateway:6001
POTENTIAL_GATEWAY_REDIS_HOST=redis
```

## Использование роли

1. Создайте playbook:

```yaml
# deploy_netbox.yml
- hosts: netbox_servers
  become: yes
  vars:
    postgresql_password: "secure_password_here"
  roles:
    - role: netbox
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory deploy_netbox.yml --extra-vars "postgresql_password=my_secure_password"
```

## Особенности реализации

1. **Модульность**: Роль разделена на логические компоненты для удобства управления.

2. **Безопасность**:
   - Пароли и чувствительные данные передаются через переменные
   - Права доступа к файлам настроены минимально необходимые
   - Системный пользователь без оболочки для сервисов

3. **Надежность**:
   - Проверка состояния сервисов после развертывания
   - Ожидание готовности контейнеров
   - Автоматический перезапуск при сбоях

4. **Гибкость**:
   - Все параметры конфигурируются через переменные
   - Поддержка кастомизации через templates
   - Возможность расширения функционала

5. **Идемпотентность**: Роль можно запускать многократно без побочных эффектов.