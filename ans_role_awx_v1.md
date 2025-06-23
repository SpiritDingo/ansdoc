# Роль Ansible для установки и настройки AWX в Docker Compose

Вот пример роли Ansible для развертывания AWX с использованием Docker Compose, с выносом конфигурации и проектов в volumes.

## Структура роли

```
roles/awx-docker/
├── defaults
│   └── main.yml
├── files
│   ├── docker-compose.yml.j2
│   └── awx.env.j2
├── tasks
│   └── main.yml
└── templates
    └── nginx.conf.j2
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки AWX
awx_version: "latest"
awx_admin_user: "admin"
awx_admin_password: "password"
awx_secret_key: "your-secret-key-here"

# Настройки PostgreSQL
postgres_data_dir: "/var/lib/awx/postgres"
postgres_image: "postgres:13"
postgres_user: "awx"
postgres_password: "awxpass"
postgres_database: "awx"
postgres_port: 5432

# Настройки Redis
redis_image: "redis:latest"

# Настройки Docker Compose
awx_project_name: "awx"
awx_data_dir: "/var/lib/awx"
awx_projects_dir: "{{ awx_data_dir }}/projects"
awx_compose_file: "/opt/awx/docker-compose.yml"
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
    - python3-pip
    - docker-compose-plugin

- name: Убедиться, что Docker запущен и включен
  service:
    name: docker
    state: started
    enabled: yes

- name: Создание директорий для volumes
  file:
    path: "{{ item }}"
    state: directory
    mode: 0755
  loop:
    - "{{ postgres_data_dir }}"
    - "{{ awx_data_dir }}"
    - "{{ awx_projects_dir }}"
    - "/opt/awx"

- name: Копирование файла окружения AWX
  template:
    src: awx.env.j2
    dest: /opt/awx/awx.env
    mode: 0640

- name: Копирование docker-compose файла
  template:
    src: docker-compose.yml.j2
    dest: "{{ awx_compose_file }}"
    mode: 0644

- name: Запуск AWX в Docker Compose
  community.docker.docker_compose:
    project_src: /opt/awx
    build: yes
    pull: yes
    recreate: always
    restart_policy: always
    state: present

- name: Ожидание готовности AWX
  uri:
    url: "http://localhost:80/api/v2/ping/"
    method: GET
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 30
  delay: 10
```

### files/docker-compose.yml.j2

```yaml
version: '3.7'
services:
  postgres:
    image: {{ postgres_image }}
    container_name: {{ awx_project_name }}_postgres
    environment:
      POSTGRES_USER: {{ postgres_user }}
      POSTGRES_PASSWORD: {{ postgres_password }}
      POSTGRES_DB: {{ postgres_database }}
    volumes:
      - {{ postgres_data_dir }}:/var/lib/postgresql/data:Z
    restart: unless-stopped
    networks:
      - awx

  redis:
    image: {{ redis_image }}
    container_name: {{ awx_project_name }}_redis
    restart: unless-stopped
    networks:
      - awx

  awx:
    image: ghcr.io/ansible/awx:{{ awx_version }}
    container_name: {{ awx_project_name }}_task
    depends_on:
      - postgres
      - redis
    hostname: awx
    user: root
    volumes:
      - {{ awx_projects_dir }}:/var/lib/awx/projects:Z
    environment:
      AWX_ADMIN_USER: {{ awx_admin_user }}
      AWX_ADMIN_PASSWORD: {{ awx_admin_password }}
      SECRET_KEY: {{ awx_secret_key }}
      DATABASE_USER: {{ postgres_user }}
      DATABASE_PASSWORD: {{ postgres_password }}
      DATABASE_NAME: {{ postgres_database }}
      DATABASE_HOST: postgres
      DATABASE_PORT: "{{ postgres_port }}"
      REDIS_HOST: redis
      REDIS_PORT: "6379"
    env_file:
      - /opt/awx/awx.env
    restart: unless-stopped
    networks:
      - awx

  awx_web:
    image: ghcr.io/ansible/awx:{{ awx_version }}
    container_name: {{ awx_project_name }}_web
    depends_on:
      - awx
    hostname: awxweb
    user: root
    ports:
      - "80:8052"
    environment:
      SECRET_KEY: {{ awx_secret_key }}
      DATABASE_USER: {{ postgres_user }}
      DATABASE_PASSWORD: {{ postgres_password }}
      DATABASE_NAME: {{ postgres_database }}
      DATABASE_HOST: postgres
      DATABASE_PORT: "{{ postgres_port }}"
      REDIS_HOST: redis
      REDIS_PORT: "6379"
      HTTP_PORT: "8052"
    env_file:
      - /opt/awx/awx.env
    restart: unless-stopped
    networks:
      - awx

networks:
  awx:
    driver: bridge
```

### files/awx.env.j2

```ini
# Настройки AWX
AWX_ADMIN_USER={{ awx_admin_user }}
AWX_ADMIN_PASSWORD={{ awx_admin_password }}
SECRET_KEY={{ awx_secret_key }}

# Настройки базы данных
DATABASE_USER={{ postgres_user }}
DATABASE_PASSWORD={{ postgres_password }}
DATABASE_NAME={{ postgres_database }}
DATABASE_HOST=postgres
DATABASE_PORT={{ postgres_port }}

# Настройки Redis
REDIS_HOST=redis
REDIS_PORT=6379

# Дополнительные настройки
AWX_SKIP_MIGRATIONS=false
AWX_ISOLATED_CHECK_INTERVAL=30
AWX_ISOLATED_LAUNCH_TIMEOUT=600
```

## Пример playbook для использования роли

```yaml
---
- name: Установка и настройка AWX в Docker
  hosts: awx_servers
  become: yes
  vars:
    awx_admin_password: "secure_password_123"
    awx_secret_key: "my-very-secret-key-1234567890"
    postgres_data_dir: "/data/awx/postgres"
    awx_data_dir: "/data/awx"
  
  roles:
    - awx-docker
```

## Примечания

1. Для работы роли необходимо установить коллекцию `community.docker`:
   ```bash
   ansible-galaxy collection install community.docker
   ```

2. Роль предполагает, что Docker и Docker Compose уже установлены на целевых хостах.

3. Все данные (PostgreSQL, проекты AWX) сохраняются в указанных volumes.

4. Для production использования:
   - Измените пароли и секретные ключи
   - Рассмотрите использование HTTPS
   - Настройте резервное копирование volumes

5. AWX будет доступен на порту 80 целевого сервера.

6. Для обновления AWX достаточно изменить переменную `awx_version` и перезапустить playbook.