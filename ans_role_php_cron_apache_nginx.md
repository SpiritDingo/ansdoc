# Ansible роль для запуска PHP скриптов через Cron в Docker Compose с PostgreSQL, Apache2/Nginx

Вот комплексная Ansible роль, которая разворачивает среду для выполнения PHP скриптов через Cron с поддержкой PostgreSQL и веб-сервером (Apache2 или Nginx на выбор).

## Структура роли

```
roles/php_web_cron/
├── tasks/
│   ├── main.yml
│   ├── apache.yml
│   └── nginx.yml
├── templates/
│   ├── docker-compose.yml.j2
│   ├── php.ini.j2
│   ├── apache.conf.j2
│   ├── nginx.conf.j2
│   ├── cron.php.j2
│   └── index.php.j2
├── files/
│   ├── wait-for-postgres.sh
│   └── healthcheck.php
└── defaults/
    └── main.yml
```

## defaults/main.yml

```yaml
---
# Настройки Docker Compose
docker_compose_dir: /opt/php_web_cron
docker_compose_file: "{{ docker_compose_dir }}/docker-compose.yml"

# Выбор веб-сервера (apache2 или nginx)
web_server: "apache2"  # или "nginx"

# Настройки PHP контейнера
php_image: php:8.2-apache  # Базовый образ с Apache
php_container_name: php_web_cron
php_volumes:
  - "{{ docker_compose_dir }}/scripts:/scripts"
  - "{{ docker_compose_dir }}/php.ini:/usr/local/etc/php/conf.d/custom.ini"
  - "{{ docker_compose_dir }}/web:/var/www/html"

# Настройки PostgreSQL
postgres_image: postgres:15
postgres_container_name: postgres_db
postgres_db: "app_db"
postgres_user: "app_user"
postgres_password: "secure_password"
postgres_port: 5432
postgres_data_dir: "{{ docker_compose_dir }}/pgdata"

# Настройки Cron
cron_entries:
  - name: "db_backup"
    schedule: "0 2 * * *"
    command: "php /scripts/db_backup.php"
    environment:
      DB_HOST: "postgres_db"
      DB_NAME: "{{ postgres_db }}"
      DB_USER: "{{ postgres_user }}"
      DB_PASSWORD: "{{ postgres_password }}"

# Настройки веб-сервера
web_port: 8080
ssl_enabled: false
ssl_cert: ""
ssl_key: ""
```

## tasks/main.yml

```yaml
---
- name: Include web server specific tasks
  ansible.builtin.include_tasks: "{{ web_server }}.yml"

- name: Ensure directories exist
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ docker_compose_dir }}"
    - "{{ docker_compose_dir }}/scripts"
    - "{{ docker_compose_dir }}/web"
    - "{{ postgres_data_dir }}"

- name: Copy PHP scripts
  ansible.builtin.template:
    src: "cron.php.j2"
    dest: "{{ docker_compose_dir }}/scripts/{{ item.name }}.php"
  loop: "{{ cron_entries }}"
  when: item.script_content is defined

- name: Deploy web index file
  ansible.builtin.template:
    src: "index.php.j2"
    dest: "{{ docker_compose_dir }}/web/index.php"
    mode: '0644'

- name: Deploy PHP configuration
  ansible.builtin.template:
    src: "php.ini.j2"
    dest: "{{ docker_compose_dir }}/php.ini"
    mode: '0644'

- name: Copy wait-for-postgres script
  ansible.builtin.copy:
    src: "wait-for-postgres.sh"
    dest: "{{ docker_compose_dir }}/wait-for-postgres.sh"
    mode: '0755'

- name: Copy healthcheck script
  ansible.builtin.copy:
    src: "healthcheck.php"
    dest: "{{ docker_compose_dir }}/web/healthcheck.php"
    mode: '0644'

- name: Deploy Docker Compose file
  ansible.builtin.template:
    src: "docker-compose.yml.j2"
    dest: "{{ docker_compose_file }}"
    mode: '0644'

- name: Ensure Docker Compose is running
  community.docker.docker_compose:
    project_src: "{{ docker_compose_dir }}"
    state: present
    restarted: yes
    recreate: always
    pull: yes

- name: Install required packages in PHP container
  ansible.builtin.command: >
    docker exec {{ php_container_name }} sh -c
    "apt-get update && apt-get install -y cron libpq-dev postgresql-client && 
     docker-php-ext-install pdo pdo_pgsql && 
     rm -rf /var/lib/apt/lists/*"
  register: install_result
  changed_when: install_result.rc == 0
  ignore_errors: yes

- name: Configure cron jobs
  ansible.builtin.command: >
    docker exec {{ php_container_name }} sh -c
    "echo '{{ item.schedule }} {{ item.environment | default({}) | dict2env }} {{ item.command }} >> /var/log/cron.log 2>&1' > /etc/cron.d/{{ item.name | regex_replace('[^A-Za-z0-9]', '_') }}"
  loop: "{{ cron_entries }}"

- name: Start cron service in container
  ansible.builtin.command: >
    docker exec {{ php_container_name }} sh -c
    "cron -f"
  async: 60
  poll: 0
```

## tasks/apache.yml

```yaml
---
- name: Configure Apache
  ansible.builtin.template:
    src: "apache.conf.j2"
    dest: "{{ docker_compose_dir }}/apache.conf"
    mode: '0644'

- name: Update PHP image for Apache
  ansible.builtin.set_fact:
    php_image: "php:8.2-apache"
```

## tasks/nginx.yml

```yaml
---
- name: Configure Nginx
  ansible.builtin.template:
    src: "nginx.conf.j2"
    dest: "{{ docker_compose_dir }}/nginx.conf"
    mode: '0644'

- name: Update PHP image for Nginx
  ansible.builtin.set_fact:
    php_image: "php:8.2-fpm"

- name: Add Nginx service to Docker Compose
  ansible.builtin.set_fact:
    additional_services: |
      nginx:
        image: nginx:latest
        container_name: nginx_web
        ports:
          - "{{ web_port }}:80"
          {% if ssl_enabled %}
          - "443:443"
          {% endif %}
        volumes:
          - {{ docker_compose_dir }}/web:/var/www/html
          - {{ docker_compose_dir }}/nginx.conf:/etc/nginx/conf.d/default.conf
        depends_on:
          - php_web_cron
```

## templates/docker-compose.yml.j2

```yaml
version: '3.8'

services:
  postgres_db:
    image: {{ postgres_image }}
    container_name: {{ postgres_container_name }}
    environment:
      POSTGRES_DB: {{ postgres_db }}
      POSTGRES_USER: {{ postgres_user }}
      POSTGRES_PASSWORD: {{ postgres_password }}
    volumes:
      - {{ postgres_data_dir }}:/var/lib/postgresql/data
    ports:
      - "{{ postgres_port }}:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{ postgres_user }} -d {{ postgres_db }}"]
      interval: 5s
      timeout: 5s
      retries: 5

  php_web_cron:
    image: {{ php_image }}
    container_name: {{ php_container_name }}
    depends_on:
      postgres_db:
        condition: service_healthy
    volumes:
      {% for volume in php_volumes %}
      - {{ volume }}
      {% endfor %}
      {% if web_server == 'apache2' %}
      - {{ docker_compose_dir }}/apache.conf:/etc/apache2/sites-available/000-default.conf
      {% endif %}
    environment:
      - DB_HOST=postgres_db
      - DB_NAME={{ postgres_db }}
      - DB_USER={{ postgres_user }}
      - DB_PASSWORD={{ postgres_password }}
    command: >
      sh -c "
      ./wait-for-postgres.sh postgres_db:5432 -- echo 'PostgreSQL is ready' &&
      {% if web_server == 'apache2' %}
      a2enmod rewrite &&
      service apache2 restart &&
      {% endif %}
      cron -f"
    restart: unless-stopped

  {{ additional_services | default('') }}
```

## templates/apache.conf.j2

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html

    <Directory /var/www/html>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

## templates/nginx.conf.j2

```nginx
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    root /var/www/html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php_web_cron:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

## templates/php.ini.j2

```ini
; Custom PHP configuration
extension=pdo_pgsql.so
extension=pgsql.so

[Date]
date.timezone = UTC

[PostgreSQL]
pgsql.ignore_notice = 0
pgsql.log_notice = 0

[Session]
session.save_handler = files
session.save_path = "/tmp"

[PHP]
post_max_size = 32M
upload_max_filesize = 32M
memory_limit = 128M
```

## templates/index.php.j2

```php
<?php
try {
    $db = new PDO(
        'pgsql:host=' . getenv('DB_HOST') . 
        ';dbname=' . getenv('DB_NAME'),
        getenv('DB_USER'),
        getenv('DB_PASSWORD')
    );
    
    $stmt = $db->query('SELECT version()');
    $version = $stmt->fetchColumn();
    
    echo "<h1>Welcome to PHP Web Cron</h1>";
    echo "<p>PostgreSQL version: " . htmlspecialchars($version) . "</p>";
    echo "<p>Cron jobs are running in the background.</p>";
} catch (PDOException $e) {
    echo "<p>Database connection failed: " . htmlspecialchars($e->getMessage()) . "</p>";
}
?>
```

## Пример использования роли

1. Создайте playbook `deploy_php_web_cron.yml`:

```yaml
---
- hosts: your_server
  become: yes
  roles:
    - php_web_cron
  vars:
    web_server: "nginx"  # или "apache2"
    cron_entries:
      - name: "db_maintenance"
        schedule: "0 4 * * *"
        command: "php /scripts/db_maintenance.php"
        script_content: |
          <?php
          try {
              $db = new PDO(
                  'pgsql:host=' . getenv('DB_HOST') . 
                  ';dbname=' . getenv('DB_NAME'),
                  getenv('DB_USER'),
                  getenv('DB_PASSWORD')
              );
              
              // Perform maintenance tasks
              $db->exec("VACUUM ANALYZE");
              error_log('Database maintenance completed');
          } catch (PDOException $e) {
              error_log('Maintenance failed: ' . $e->getMessage());
          }
```

2. Запустите playbook:

```bash
ansible-playbook deploy_php_web_cron.yml
```

## Особенности реализации

1. **Поддержка двух веб-серверов**:
   - Apache2 с mod_php
   - Nginx с PHP-FPM

2. **Полноценная веб-среда**:
   - Рабочий веб-интерфейс
   - Healthcheck-страница
   - Поддержка SSL (нужно добавить сертификаты)

3. **Интеграция с PostgreSQL**:
   - Автоматическая установка драйверов
   - Проверка подключения к БД
   - Примеры работы с PDO

4. **Гибкая настройка**:
   - Выбор веб-сервера через переменную
   - Настраиваемые порты
   - Возможность добавления SSL

5. **Мониторинг**:
   - Логирование выполнения cron-задач
   - Healthcheck для PostgreSQL
   - Веб-интерфейс для проверки состояния

Эта роль предоставляет готовое решение для развертывания веб-приложения с фоновыми задачами, работающими через cron, и интеграцией с PostgreSQL.