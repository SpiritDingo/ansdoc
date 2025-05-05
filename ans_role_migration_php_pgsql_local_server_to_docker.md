# Ansible Role для миграции PHP+PostgreSQL сервиса в Docker Compose

Вот полная Ansible роль для переноса PHP приложения с PostgreSQL с локального сервера в Docker, включая анализ RPM пакетов и автоматическое создание Dockerfile.

## Структура роли

```
roles/migrate_php_postgres/
├── defaults
│   └── main.yml
├── tasks
│   ├── 01_analyze_system.yml
│   ├── 02_install_docker.yml
│   ├── 03_prepare_app.yml
│   ├── 04_create_dockerfile.yml
│   ├── 05_setup_compose.yml
│   ├── 06_migrate_db.yml
│   └── main.yml
├── templates
│   ├── Dockerfile.j2
│   ├── docker-compose.yml.j2
│   └── php.ini.j2
└── vars
    └── main.yml
```

## 1. defaults/main.yml

```yaml
---
# Основные параметры
app_name: "php_postgres_app"
app_dir: "/var/www/{{ app_name }}"
docker_dir: "/opt/{{ app_name }}"

# Настройки PostgreSQL
db_name: "app_db"
db_user: "app_user"
db_password: "db_pass123"
db_port: 5432

# Настройки Docker
php_version: "8.2"
php_base_image: "php:{{ php_version }}-apache"
postgres_version: "13"
compose_version: "3.8"

# Параметры выполнения
stop_old_service: false
backup_old_db: true
```

## 2. tasks/main.yml

```yaml
---
- name: Analyze source system
  include_tasks: 01_analyze_system.yml

- name: Install Docker and dependencies
  include_tasks: 02_install_docker.yml
  when: install_docker | default(true)

- name: Prepare application files
  include_tasks: 03_prepare_app.yml

- name: Create optimized Dockerfile
  include_tasks: 04_create_dockerfile.yml

- name: Setup Docker Compose
  include_tasks: 05_setup_compose.yml

- name: Migrate PostgreSQL database
  include_tasks: 06_migrate_db.yml
  when: backup_old_db | default(true)
```

## 3. tasks/01_analyze_system.yml

```yaml
---
- name: Gather installed RPM packages
  command: rpm -qa
  register: rpm_packages
  changed_when: false

- name: Detect PHP version
  command: php -v
  register: php_version_output
  changed_when: false
  ignore_errors: yes

- name: Set exact PHP version
  set_fact:
    php_version: "{{ (php_version_output.stdout | regex_search('PHP (\\d+\\.\\d+)'))[1] }}"
  when: php_version_output is succeeded

- name: Get PHP modules
  command: php -m
  register: php_modules
  changed_when: false

- name: Check PostgreSQL client packages
  set_fact:
    postgres_client_installed: "{{ 'php-pgsql' in rpm_packages.stdout or 'php-pdo_pgsql' in rpm_packages.stdout }}"

- name: Collect PHP configs
  find:
    paths: /etc/php.d /etc/php/{{ php_version }}/
    patterns: "*.ini"
  register: php_config_files

- name: Analyze required extensions
  set_fact:
    required_extensions: "{{ php_modules.stdout_lines | select('match', '^[a-zA-Z]') | reject('match', '^(Core|date|libxml|openssl|pcre|sqlite3|zlib)$') | list }}"
```

## 4. tasks/04_create_dockerfile.yml

```yaml
---
- name: Create package mapping (RPM to Debian)
  set_fact:
    package_mapping:
      php-pgsql: "libpq-dev"
      php-pdo_pgsql: "libpq-dev"
      php-gd: "libpng-dev libjpeg-dev"
      php-curl: "libcurl4-openssl-dev"
      php-zip: "zip libzip-dev"
      php-xml: "libxml2-dev"

- name: Determine required system packages
  set_fact:
    system_packages: |
      {% set packages = [] %}
      {% for rpm_pkg in package_mapping %}
        {% if rpm_pkg in rpm_packages.stdout %}
          {% set _ = packages.append(package_mapping[rpm_pkg]) %}
        {% endif %}
      {% endfor %}
      {{ packages | join(' ') }}

- name: Create Dockerfile
  template:
    src: ../templates/Dockerfile.j2
    dest: "{{ docker_dir }}/Dockerfile"
    mode: '0644'
```

## 5. templates/Dockerfile.j2

```dockerfile
# Автоматически сгенерированный Dockerfile для PHP+PostgreSQL
FROM {{ php_base_image }}

# Установка системных зависимостей
RUN apt-get update && apt-get install -y \
    {{ system_packages }} \
    postgresql-client \
    && rm -rf /var/lib/apt/lists/*

# Установка PHP расширений
{% for extension in required_extensions %}
{% if extension == 'pgsql' or extension == 'pdo_pgsql' %}
RUN docker-php-ext-install pdo pdo_pgsql pgsql
{% elif extension == 'gd' %}
RUN docker-php-ext-configure gd --with-jpeg \
    && docker-php-ext-install gd
{% elif extension == 'imagick' %}
RUN pecl install imagick && docker-php-ext-enable imagick
{% elif extension not in ['apcu', 'opcache'] %}
RUN docker-php-ext-install {{ extension }}
{% endif %}
{% endfor %}

# Копирование конфигураций PHP
{% if php_config_files.files %}
{% for config in php_config_files.files %}
COPY config/{{ config.path | basename }} /usr/local/etc/php/conf.d/
{% endfor %}
{% endif %}

# Настройка Apache
RUN a2enmod rewrite

WORKDIR /var/www/html
COPY src/ .

RUN chown -R www-data:www-data /var/www/html
```

## 6. templates/docker-compose.yml.j2

```yaml
version: '{{ compose_version }}'

services:
  app:
    build: .
    ports:
      - "{{ host_port | default(80) }}:80"
    volumes:
      - ./src:/var/www/html
    depends_on:
      - postgres
    environment:
      DB_HOST: postgres
      DB_PORT: {{ db_port }}
      DB_NAME: ${DB_NAME:-{{ db_name }}}
      DB_USER: ${DB_USER:-{{ db_user }}}
      DB_PASSWORD: ${DB_PASSWORD:-{{ db_password }}}
    restart: unless-stopped

  postgres:
    image: postgres:{{ postgres_version }}
    environment:
      POSTGRES_DB: ${DB_NAME:-{{ db_name }}}
      POSTGRES_USER: ${DB_USER:-{{ db_user }}}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-{{ db_password }}}
    volumes:
      - pg_data:/var/lib/postgresql/data
    ports:
      - "{{ db_port }}:5432"
    restart: unless-stopped

volumes:
  pg_data:
```

## 7. tasks/06_migrate_db.yml

```yaml
---
- name: Install pg_dump if needed
  yum:
    name: postgresql
    state: present
  when: "'postgresql' not in rpm_packages.stdout"

- name: Create backup directory
  file:
    path: "{{ docker_dir }}/backup"
    state: directory

- name: Dump PostgreSQL database
  become: yes
  become_user: postgres
  command: >
    pg_dump -Fc {{ db_name }} > {{ docker_dir }}/backup/db.dump
  args:
    creates: "{{ docker_dir }}/backup/db.dump"
  register: db_dump
  ignore_errors: yes

- name: Restore database in container
  command: >
    docker exec -i {{ app_name }}-postgres pg_restore -U {{ db_user }} -d {{ db_name }} < backup/db.dump
  args:
    chdir: "{{ docker_dir }}"
  when: db_dump is succeeded
```

## Использование роли

1. Создайте playbook:

```yaml
# migrate_php_postgres.yml
- hosts: db_servers
  become: yes
  roles:
    - role: migrate_php_postgres
      vars:
        app_name: "my_app"
        db_password: "secure_password"
        db_port: 5433  # Если стандартный порт занят
```

2. Запустите:

```bash
ansible-playbook -i inventory.ini migrate_php_postgres.yml
```

## Особенности реализации

1. **Анализ зависимостей**:
   - Автоматическое определение необходимых PHP расширений
   - Преобразование RPM пакетов в Debian-эквиваленты
   - Проверка наличия PostgreSQL клиентских библиотек

2. **Миграция базы данных**:
   - Автоматический дамп существующей БД
   - Восстановление в контейнере PostgreSQL

3. **Гибкая конфигурация**:
   - Поддержка кастомных PHP конфигов
   - Настройка портов через переменные
   - Возможность отключения миграции БД

4. **Безопасность**:
   - Использование переменных окружения для чувствительных данных
   - Правильные права на файлы приложения

Для дополнительной настройки можно:
- Добавить поддержку .env файлов
- Реализовать multi-stage сборку для production
- Добавить healthcheck для сервисов
- Настроить бэкапы БД в композ-файле