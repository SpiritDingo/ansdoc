# Ansible Role для миграции PHP сервиса в Docker Compose с анализом RPM

Создадим полноценную Ansible роль, которая анализирует текущий PHP сервис, собирает информацию об установленных пакетах и автоматически создает оптимальную Docker-конфигурацию.

## Структура роли

```
roles/migrate_php_to_docker/
├── defaults
│   └── main.yml
├── tasks
│   ├── analyze_system.yml
│   ├── install_docker.yml
│   ├── prepare_application.yml
│   ├── create_dockerfile.yml
│   ├── setup_compose.yml
│   └── main.yml
├── templates
│   ├── Dockerfile.j2
│   └── docker-compose.yml.j2
└── vars
    └── main.yml
```

## 1. defaults/main.yml

```yaml
---
# Основные параметры приложения
app_name: "my_php_app"
app_dir: "/var/www/{{ app_name }}"
docker_dir: "/opt/{{ app_name }}"

# Параметры базы данных
db_name: "app_db"
db_user: "app_user"
db_password: "changeme"
db_root_password: "rootpass"

# Настройки Docker
docker_compose_version: "3.8"
php_base_image: "php:8.2-apache"

# Контроль выполнения
stop_old_service: false
migrate_database: true
```

## 2. tasks/main.yml

```yaml
---
- name: Include system analysis tasks
  include_tasks: analyze_system.yml

- name: Install Docker and dependencies
  include_tasks: install_docker.yml
  when: install_docker | default(true)

- name: Prepare application files
  include_tasks: prepare_application.yml

- name: Create optimized Dockerfile
  include_tasks: create_dockerfile.yml

- name: Setup Docker Compose
  include_tasks: setup_compose.yml

- name: Migrate database if needed
  include_tasks: migrate_database.yml
  when: migrate_database | default(false)
```

## 3. tasks/analyze_system.yml

```yaml
---
- name: Gather installed RPM packages
  command: rpm -qa
  register: rpm_packages
  changed_when: false

- name: Get PHP version
  command: php -v
  register: php_version_output
  changed_when: false
  ignore_errors: yes

- name: Extract PHP version
  set_fact:
    php_version: "{{ (php_version_output.stdout | regex_search('PHP (\\d+\\.\\d+)'))[1] }}"
  when: php_version_output is succeeded

- name: Get PHP modules
  command: php -m
  register: php_modules
  changed_when: false

- name: Detect web server
  set_fact:
    web_server: "{{ 'apache' if ('httpd' in rpm_packages.stdout) else 'nginx' }}"

- name: Collect PHP configuration files
  find:
    paths: /etc/php.d /etc/php/{{ php_version }}/
    patterns: "*.ini"
  register: php_config_files

- name: Analyze required extensions
  set_fact:
    required_extensions: "{{ php_modules.stdout_lines | select('match', '^[a-zA-Z]') | reject('match', '^(Core|date|libxml|openssl|pcre|sqlite3|zlib)$') | list }}"
```

## 4. tasks/create_dockerfile.yml

```yaml
---
- name: Create mapping for RPM to Debian packages
  set_fact:
    package_mapping:
      php-mysql: "php-mysqlnd"
      php-gd: "libpng-dev libjpeg-dev"
      php-curl: "libcurl4-openssl-dev"
      php-zip: "zip libzip-dev"
      php-xml: "libxml2-dev"
      php-mbstring: "libonig-dev"
      php-imagick: "libmagickwand-dev"

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

- name: Create Dockerfile from template
  template:
    src: ../templates/Dockerfile.j2
    dest: "{{ docker_dir }}/Dockerfile"
  vars:
    php_extensions: "{{ required_extensions }}"
    system_packages: "{{ system_packages }}"
```

## 5. templates/Dockerfile.j2

```dockerfile
# Автоматически сгенерированный Dockerfile
FROM {{ php_base_image }}

# Установка системных зависимостей
RUN apt-get update && apt-get install -y \
    {{ system_packages }} \
    && rm -rf /var/lib/apt/lists/*

# Установка PHP расширений
{% for extension in php_extensions %}
{% if extension == 'gd' %}
RUN docker-php-ext-configure gd --with-jpeg \
    && docker-php-ext-install gd
{% elif extension == 'imagick' %}
RUN pecl install imagick && docker-php-ext-enable imagick
{% elif extension == 'redis' %}
RUN pecl install redis && docker-php-ext-enable redis
{% elif extension not in ['apcu', 'opcache'] %}
RUN docker-php-ext-install {{ extension }}
{% endif %}
{% endfor %}

# Копирование конфигураций PHP
{% if php_config_files.files %}
{% for config in php_config_files.files %}
COPY {{ config.path }} /usr/local/etc/php/conf.d/
{% endfor %}
{% endif %}

# Настройка веб-сервера
{% if 'apache' in web_server %}
RUN a2enmod rewrite
{% endif %}

WORKDIR /var/www/html
COPY src/ .

RUN chown -R www-data:www-data /var/www/html
```

## 6. templates/docker-compose.yml.j2

```yaml
version: '{{ docker_compose_version }}'

services:
  app:
    build: .
    ports:
      - "{{ host_port | default(80) }}:80"
    volumes:
      - ./src:/var/www/html
    {% if 'pdo_mysql' in php_extensions or 'mysqli' in php_extensions %}
    depends_on:
      - db
    environment:
      DB_HOST: db
      DB_NAME: ${DB_NAME:-{{ db_name }}}
      DB_USER: ${DB_USER:-{{ db_user }}}
      DB_PASSWORD: ${DB_PASSWORD:-{{ db_password }}}
    {% endif %}
    restart: unless-stopped

  {% if 'pdo_mysql' in php_extensions or 'mysqli' in php_extensions %}
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD:-{{ db_root_password }}}
      MYSQL_DATABASE: ${DB_NAME:-{{ db_name }}}
      MYSQL_USER: ${DB_USER:-{{ db_user }}}
      MYSQL_PASSWORD: ${DB_PASSWORD:-{{ db_password }}}
    volumes:
      - db_data:/var/lib/mysql
    ports:
      - "{{ db_port | default('3306') }}:3306"
    restart: unless-stopped
  {% endif %}

volumes:
  {% if 'pdo_mysql' in php_extensions or 'mysqli' in php_extensions %}
  db_data:
  {% endif %}
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
# migrate_php.yml
- hosts: target_servers
  become: yes
  roles:
    - role: migrate_php_to_docker
      vars:
        app_name: "my_custom_app"
        app_dir: "/var/www/custom_app"
        db_password: "secure_password_123"
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini migrate_php.yml
```

## Дополнительные возможности

1. **Миграция базы данных** - добавьте задачу в `tasks/migrate_database.yml`
2. **Настройка окружения** - добавьте обработку .env файлов
3. **Оптимизация образов** - добавьте multi-stage build для production
4. **Поддержка разных ОС** - расширьте анализ пакетов для разных дистрибутивов

Эта роль обеспечивает полный цикл миграции PHP приложения с анализом текущей системы и автоматическим созданием оптимальной Docker-конфигурации.