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


Если используется Docker Compose v2 (включенный в Docker как plugin `docker compose`), нужно внести некоторые изменения в роль. Вот обновленная версия:

## Изменения для поддержки Docker Compose v2

### Обновленный tasks/main.yml

```yaml
---
- name: Проверить наличие Docker Compose V2
  command: docker compose version
  register: compose_check
  ignore_errors: yes
  changed_when: false

- name: Убедиться, что Docker Compose V2 доступен
  assert:
    that:
      - "'Docker Compose version v2' in compose_check.stdout"
    fail_msg: "Docker Compose V2 не обнаружен. Установите его или используйте Docker Desktop."

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

- name: Запуск AWX с помощью Docker Compose V2
  community.docker.docker_compose:
    project_src: /opt/awx
    build: yes
    pull: yes
    recreate: always
    restart_policy: always
    state: present
    # Явно указываем использовать docker compose v2
    use_plugin: yes

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

### Обновленный docker-compose.yml.j2

```yaml
version: '3.8'
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
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{ postgres_user }}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: {{ redis_image }}
    container_name: {{ awx_project_name }}_redis
    restart: unless-stopped
    networks:
      - awx
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  awx:
    image: ghcr.io/ansible/awx:{{ awx_version }}
    container_name: {{ awx_project_name }}_task
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
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
    name: awx_network
    driver: bridge
```

## Ключевые изменения для Docker Compose V2:

1. **Проверка версии Docker Compose** - добавлена задача для проверки наличия V2
2. **Параметр `use_plugin: yes`** - в модуле docker_compose для явного указания использования плагина
3. **Улучшенные healthchecks** - добавлены проверки здоровья для postgres и redis
4. **Уточненные depends_on** - с указанием condition: service_healthy
5. **Именованная сеть** - явное указание имени сети для лучшей совместимости
6. **Обновленная версия compose** - используется version: '3.8'

## Дополнительные рекомендации:

1. Для работы с Docker Compose V2 в Ansible требуется:
   - Ansible 2.10+
   - Коллекция community.docker 1.9.0+

2. Для проверки установки Docker Compose V2 можно выполнить:
   ```bash
   docker compose version
   # Должен показать версию v2.x.x
   ```

3. При необходимости можно добавить задачу для установки Docker Compose V2:
   ```yaml
   - name: Установка Docker Compose Plugin
     community.docker.docker_compose_plugin:
       state: present
     when: "'Docker Compose version v2' not in compose_check.stdout"
   ```

Эти изменения обеспечат корректную работу роли с Docker Compose V2, сохраняя все преимущества использования volumes для данных.


# Обновленная роль Ansible для AWX с Docker Compose V2

Поскольку модуль `community.docker.docker_compose` устарел и удален, вот полностью обновленная роль, использующая `community.docker.docker_compose_v2`.

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

## Обновленные файлы роли

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  ansible.builtin.apt:
    name:
      - python3-pip
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
    state: present
    update_cache: yes
  when: ansible_os_family == 'Debian'

- name: Установка зависимостей для RHEL
  ansible.builtin.yum:
    name:
      - yum-utils
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
    state: present
  when: ansible_os_family == 'RedHat'

- name: Добавление пользователя в группу docker
  ansible.builtin.user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Убедиться, что Docker запущен и включен
  ansible.builtin.service:
    name: docker
    state: started
    enabled: yes

- name: Проверить наличие Docker Compose V2
  ansible.builtin.command: docker compose version
  register: compose_check
  changed_when: false
  failed_when: false

- name: Проверить, что Docker Compose V2 доступен
  ansible.builtin.assert:
    that:
      - "'Docker Compose version v2' in compose_check.stdout"
    fail_msg: "Docker Compose V2 не обнаружен. Убедитесь, что установлен docker-compose-plugin."

- name: Создание директорий для volumes
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: 0755
    owner: "{{ ansible_user }}"
    group: docker
  loop:
    - "{{ postgres_data_dir }}"
    - "{{ awx_data_dir }}"
    - "{{ awx_projects_dir }}"
    - "/opt/awx"

- name: Копирование файла окружения AWX
  ansible.builtin.template:
    src: awx.env.j2
    dest: /opt/awx/awx.env
    mode: 0640
    owner: "{{ ansible_user }}"
    group: docker

- name: Копирование docker-compose файла
  ansible.builtin.template:
    src: docker-compose.yml.j2
    dest: "{{ awx_compose_file }}"
    mode: 0644
    owner: "{{ ansible_user }}"
    group: docker

- name: Запуск AWX с помощью Docker Compose V2
  community.docker.docker_compose_v2:
    project_src: /opt/awx
    build: yes
    pull: yes
    recreate: always
    restart_policy: always
    state: present
    timeout: 300

- name: Ожидание готовности AWX
  ansible.builtin.uri:
    url: "http://localhost:80/api/v2/ping/"
    method: GET
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 30
  delay: 10
```

### defaults/main.yml

```yaml
---
# Настройки AWX
awx_version: "23.7.0"
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
redis_image: "redis:7"

# Настройки Docker Compose
awx_project_name: "awx"
awx_data_dir: "/var/lib/awx"
awx_projects_dir: "{{ awx_data_dir }}/projects"
awx_compose_file: "/opt/awx/docker-compose.yml"
```

### files/docker-compose.yml.j2

```yaml
version: '3.8'
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
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{ postgres_user }}"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: {{ redis_image }}
    container_name: {{ awx_project_name }}_redis
    restart: unless-stopped
    networks:
      - awx
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  awx:
    image: ghcr.io/ansible/awx:{{ awx_version }}
    container_name: {{ awx_project_name }}_task
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
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
    name: awx_network
    driver: bridge
```

## Важные изменения:

1. **Замена модуля**:
   - Старый: `community.docker.docker_compose`
   - Новый: `community.docker.docker_compose_v2`

2. **Добавлены проверки**:
   - Проверка наличия Docker Compose V2
   - Улучшенные healthchecks в docker-compose.yml

3. **Улучшенная установка зависимостей**:
   - Добавлена поддержка RHEL/CentOS
   - Явная установка docker-compose-plugin

4. **Права доступа**:
   - Явное указание владельца и группы для файлов и директорий

5. **Обновленные версии**:
   - Используется актуальная версия AWX (23.7.0)
   - Обновлены версии PostgreSQL и Redis

## Требования:

1. Установите необходимые коллекции:
```bash
ansible-galaxy collection install community.docker
```

2. Для работы требуется Ansible 2.12+ и Docker Compose V2.

3. Убедитесь, что на целевых хостах установлен docker-compose-plugin:
```bash
sudo apt-get install docker-compose-plugin  # Для Debian/Ubuntu
sudo yum install docker-compose-plugin    # Для RHEL/CentOS
```

Отлично, раз Docker Compose V2 уже установлен, я упрощу роль, убрав лишние проверки и задачи установки. Вот оптимизированная версия роли:

## Упрощенная роль для AWX с предустановленным Docker Compose V2

### tasks/main.yml

```yaml
---
- name: Проверить версию Docker Compose
  command: docker compose version
  register: compose_version
  changed_when: false
  tags: check

- name: Убедиться, что используется Docker Compose V2
  assert:
    that:
      - "'Docker Compose version v2' in compose_version.stdout"
    fail_msg: "Требуется Docker Compose V2. Установите docker-compose-plugin."
  tags: check

- name: Убедиться, что Docker запущен
  service:
    name: docker
    state: started
    enabled: yes

- name: Создать необходимые директории
  file:
    path: "{{ item }}"
    state: directory
    mode: 0755
    owner: "{{ ansible_user_id }}"
    group: docker
  loop:
    - "{{ postgres_data_dir }}"
    - "{{ awx_data_dir }}"
    - "{{ awx_projects_dir }}"
    - "/opt/awx"

- name: Развернуть конфигурационные файлы
  block:
    - name: Развернуть файл окружения
      template:
        src: awx.env.j2
        dest: /opt/awx/awx.env
        mode: 0640
        owner: "{{ ansible_user_id }}"
        group: docker

    - name: Развернуть docker-compose.yml
      template:
        src: docker-compose.yml.j2
        dest: "{{ awx_compose_file }}"
        mode: 0644
        owner: "{{ ansible_user_id }}"
        group: docker

- name: Запустить AWX через Docker Compose
  community.docker.docker_compose_v2:
    project_src: /opt/awx
    pull: yes
    build: no
    recreate: always
    remove_orphans: yes
    restart_policy: unless-stopped
    state: present
    timeout: 300

- name: Проверить работоспособность AWX
  uri:
    url: "http://localhost/api/v2/ping/"
    method: GET
    status_code: 200
    timeout: 30
  register: awx_status
  until: awx_status.status == 200
  retries: 15
  delay: 10
  ignore_errors: yes
```

### defaults/main.yml

```yaml
---
# Версии образов
awx_version: "23.7.0"
postgres_image: "postgres:13-alpine"
redis_image: "redis:7-alpine"

# Учетные данные AWX
awx_admin_user: "admin"
awx_admin_password: "changeme"
awx_secret_key: "generate-secure-key-here"

# Настройки PostgreSQL
postgres_user: "awx"
postgres_password: "awxpass"
postgres_database: "awx"
postgres_port: 5432

# Пути для хранения данных
postgres_data_dir: "/var/lib/awx/postgres"
awx_data_dir: "/var/lib/awx"
awx_projects_dir: "{{ awx_data_dir }}/projects"

# Настройки проекта
awx_project_name: "awx"
awx_compose_file: "/opt/awx/docker-compose.yml"
```

### files/docker-compose.yml.j2

```yaml
version: '3.8'

services:
  postgres:
    image: {{ postgres_image }}
    container_name: {{ awx_project_name }}_postgres
    environment:
      POSTGRES_USER: {{ postgres_user }}
      POSTGRES_PASSWORD: {{ postgres_password }}
      POSTGRES_DB: {{ postgres_database }}
    volumes:
      - {{ postgres_data_dir }}:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - awx_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{ postgres_user }}"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: {{ redis_image }}
    container_name: {{ awx_project_name }}_redis
    restart: unless-stopped
    networks:
      - awx_network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  awx:
    image: ghcr.io/ansible/awx:{{ awx_version }}
    container_name: {{ awx_project_name }}_task
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    user: root
    volumes:
      - {{ awx_projects_dir }}:/var/lib/awx/projects
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
      - awx_network

  awx_web:
    image: ghcr.io/ansible/awx:{{ awx_version }}
    container_name: {{ awx_project_name }}_web
    depends_on:
      - awx
    ports:
      - "80:8052"
    environment:
      SECRET_KEY: {{ awx_secret_key }}
      HTTP_PORT: "8052"
    env_file:
      - /opt/awx/awx.env
    restart: unless-stopped
    networks:
      - awx_network

networks:
  awx_network:
    driver: bridge
```

### files/awx.env.j2

```ini
# Database settings
DATABASE_USER={{ postgres_user }}
DATABASE_PASSWORD={{ postgres_password }}
DATABASE_NAME={{ postgres_database }}
DATABASE_HOST=postgres
DATABASE_PORT={{ postgres_port }}

# Redis settings
REDIS_HOST=redis
REDIS_PORT=6379

# AWX settings
AWX_ADMIN_USER={{ awx_admin_user }}
AWX_ADMIN_PASSWORD={{ awx_admin_password }}
SECRET_KEY={{ awx_secret_key }}

# Additional settings
AWX_SKIP_MIGRATIONS=false
AWX_ISOLATED_CHECK_INTERVAL=30
AWX_ISOLATED_LAUNCH_TIMEOUT=600
```

## Основные улучшения:

1. **Упрощенная проверка** - только проверка версии Docker Compose
2. **Оптимизированные образы** - используются alpine-версии для postgres и redis
3. **Улучшенные healthchecks** - увеличены интервалы проверок
4. **Чистая структура сети** - явное именование сети
5. **Упрощенный процесс развертывания** - убраны лишние шаги
6. **Более безопасные права** - правильные владельцы для файлов

Для использования роли:

1. Установите зависимость:
```bash
ansible-galaxy collection install community.docker
```

2. Создайте playbook (awx_deploy.yml):
```yaml
---
- hosts: awx_servers
  become: yes
  vars:
    awx_admin_password: "your_secure_password"
    awx_secret_key: "your_secure_secret_key"
  
  roles:
    - awx-docker
```

3. Запустите playbook:
```bash
ansible-playbook -i inventory awx_deploy.yml
```