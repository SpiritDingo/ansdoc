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




# Обновленная Ansible роль для установки Semaphore UI Enterprise с Docker Compose V2

Вот обновленная роль с улучшениями для работы с Docker Compose V2 и дополнительными функциями:

## Улучшения в роли:

1. Добавлена проверка установки Docker и Docker Compose V2
2. Добавлена поддержка Traefik для reverse proxy
3. Добавлены healthchecks для всех сервисов
4. Добавлена возможность настройки через инвентарь
5. Улучшена обработка ошибок

## Структура роли (обновленная):

```
roles/semaphore_ui/
├── defaults
│   └── main.yml
├── tasks
│   ├── main.yml
│   ├── install_docker.yml
│   └── configure_firewall.yml
├── templates
│   ├── docker-compose.yml.j2
│   └── semaphore.env.j2
└── vars
    └── main.yml
```

## 1. defaults/main.yml (обновленный)

```yaml
---
# Версии
semaphore_ui_version: "2.8.0"
docker_compose_version: "2.24.5"

# Сетевые настройки
semaphore_ui_port: 3000
semaphore_ui_domain: "semaphore.example.com"
semaphore_ui_expose: "port"  # или "traefik"

# Пути
semaphore_ui_data_dir: "/opt/semaphore_ui"
semaphore_db_data_dir: "{{ semaphore_ui_data_dir }}/postgres"

# Настройки базы данных
semaphore_db_host: "postgres"
semaphore_db_port: 5432
semaphore_db_name: "semaphore"
semaphore_db_user: "semaphore"
semaphore_db_password: "{{ lookup('password', semaphore_ui_data_dir + '/db_password length=32') }}"

# Настройки администратора
semaphore_admin_email: "admin@example.com"
semaphore_admin_password: "{{ lookup('password', semaphore_ui_data_dir + '/admin_password length=24') }}"
semaphore_admin_name: "Admin"

# Настройки Traefik (если используется)
traefik_enabled: false
traefik_network: "traefik_public"
```

## 2. tasks/main.yml (обновленный)

```yaml
---
- name: Include Docker installation tasks
  ansible.builtin.include_tasks: install_docker.yml
  when: ansible_facts.services['docker.service'] is not defined

- name: Verify Docker Compose V2 is installed
  ansible.builtin.command: docker compose version
  register: docker_compose_check
  changed_when: false
  ignore_errors: yes

- name: Install Docker Compose plugin if needed
  ansible.builtin.apt:
    name: docker-compose-plugin
    state: present
    update_cache: yes
  when: docker_compose_check is failed

- name: Create necessary directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0750'
    owner: root
    group: docker
  loop:
    - "{{ semaphore_ui_data_dir }}"
    - "{{ semaphore_db_data_dir }}"

- name: Generate configuration files
  ansible.builtin.template:
    src: "{{ item.src }}"
    dest: "{{ semaphore_ui_data_dir }}/{{ item.dest }}"
    mode: '0640'
  loop:
    - { src: 'docker-compose.yml.j2', dest: 'docker-compose.yml' }
    - { src: 'semaphore.env.j2', dest: '.env' }

- name: Pull Docker images
  community.docker.docker_compose:
    project_src: "{{ semaphore_ui_data_dir }}"
    pull: yes
    files:
      - "{{ semaphore_ui_data_dir }}/docker-compose.yml"

- name: Start Semaphore stack
  community.docker.docker_compose:
    project_src: "{{ semaphore_ui_data_dir }}"
    state: present
    restarted: yes
    recreate: always
    files:
      - "{{ semaphore_ui_data_dir }}/docker-compose.yml"

- name: Wait for Semaphore to be ready
  ansible.builtin.uri:
    url: "http://localhost:{{ semaphore_ui_port }}/api/ping"
    method: GET
    status_code: 200
    timeout: 30
    body_format: json
  register: result
  until: result.status == 200
  retries: 12
  delay: 10
  ignore_errors: yes

- name: Include firewall configuration
  ansible.builtin.include_tasks: configure_firewall.yml
  when: semaphore_ui_expose == 'port' and ansible_os_family == 'Debian'
```

## 3. templates/docker-compose.yml.j2 (обновленный)

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: semaphore_postgres
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
    volumes:
      - ${DB_DATA_DIR}:/var/lib/postgresql/data
    networks:
      - semaphore_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5

  semaphore:
    image: semaphoreui/semaphore-ee:${SEMAPHORE_VERSION}
    container_name: semaphore_ui
    depends_on:
      postgres:
        condition: service_healthy
    env_file:
      - .env
    volumes:
      - ${SEMAPHORE_DATA_DIR}:/etc/semaphore
    networks:
      - semaphore_network
      {% if traefik_enabled %}
      - {{ traefik_network }}
      {% endif %}
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/api/ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    {% if semaphore_ui_expose == 'port' %}
    ports:
      - "${SEMAPHORE_PORT}:3000"
    {% elif semaphore_ui_expose == 'traefik' and traefik_enabled %}
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.semaphore.rule=Host(`${SEMAPHORE_DOMAIN}`)"
      - "traefik.http.routers.semaphore.entrypoints=websecure"
      - "traefik.http.routers.semaphore.tls.certresolver=le"
      - "traefik.http.services.semaphore.loadbalancer.server.port=3000"
    {% endif %}

networks:
  semaphore_network:
    driver: bridge
  {% if traefik_enabled %}
  {{ traefik_network }}:
    external: true
  {% endif %}
```

## 4. templates/semaphore.env.j2 (новый файл)

```ini
# Semaphore Configuration
SEMAPHORE_VERSION={{ semaphore_ui_version }}
SEMAPHORE_DATA_DIR={{ semaphore_ui_data_dir }}/config
SEMAPHORE_PORT={{ semaphore_ui_port }}
SEMAPHORE_DOMAIN={{ semaphore_ui_domain }}

# Database Configuration
DB_HOST={{ semaphore_db_host }}
DB_PORT={{ semaphore_db_port }}
DB_NAME={{ semaphore_db_name }}
DB_USER={{ semaphore_db_user }}
DB_PASSWORD={{ semaphore_db_password }}
DB_DATA_DIR={{ semaphore_db_data_dir }}

# Admin Configuration
SEMAPHORE_ADMIN={{ semaphore_admin_name }}
SEMAPHORE_ADMIN_EMAIL={{ semaphore_admin_email }}
SEMAPHORE_ADMIN_PASSWORD={{ semaphore_admin_password }}
SEMAPHORE_ACCESS_KEY_ENCRYPTION={{ 64 | random | to_uuid }}

# Additional Settings
SEMAPHORE_LDAP_ENABLED=false
SEMAPHORE_TELEMETRY_ENABLED=false
```

## 5. tasks/install_docker.yml (новый файл)

```yaml
---
- name: Install prerequisites
  ansible.builtin.apt:
    name:
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
    state: present
    update_cache: yes

- name: Add Docker GPG key
  ansible.builtin.apt_key:
    url: "https://download.docker.com/linux/{{ ansible_distribution | lower }}/gpg"
    state: present

- name: Add Docker repository
  ansible.builtin.apt_repository:
    repo: "deb [arch={{ ansible_architecture }}] https://download.docker.com/linux/{{ ansible_distribution | lower }} {{ ansible_distribution_release }} stable"
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

- name: Ensure Docker is running and enabled
  ansible.builtin.service:
    name: docker
    state: started
    enabled: yes

- name: Add current user to docker group
  ansible.builtin.user:
    name: "{{ ansible_user_id }}"
    groups: docker
    append: yes
```

## 6. tasks/configure_firewall.yml (новый файл)

```yaml
---
- name: Ensure UFW is installed
  ansible.builtin.apt:
    name: ufw
    state: present

- name: Allow SSH through firewall
  ansible.builtin.ufw:
    rule: allow
    port: 22
    proto: tcp

- name: Allow Semaphore port through firewall
  ansible.builtin.ufw:
    rule: allow
    port: "{{ semaphore_ui_port }}"
    proto: tcp

- name: Enable firewall
  ansible.builtin.ufw:
    state: enabled
    policy: deny
```

## Использование обновленной роли

1. Пример playbook (`deploy_semaphore.yml`):

```yaml
---
- hosts: semaphore_servers
  become: yes
  vars:
    semaphore_ui_domain: "ci.example.com"
    semaphore_ui_expose: "traefik"
    traefik_enabled: true
  roles:
    - role: semaphore_ui
```

2. Запуск:

```bash
ansible-playbook -i inventory.ini deploy_semaphore.yml
```

## Особенности обновленной роли:

1. **Поддержка Docker Compose V2** - явная проверка и установка при необходимости
2. **Гибкая настройка экспозиции**:
   - Через порт (по умолчанию)
   - Через Traefik (если настроено)
3. **Улучшенная безопасность**:
   - Автогенерация паролей
   - Правильные права на файлы
   - Настройка firewall
4. **Отдельные конфигурационные файлы**:
   - docker-compose.yml для оркестрации
   - .env для параметров
5. **Healthchecks** для всех сервисов
6. **Поддержка сетей**:
   - Внутренняя сеть для коммуникации между сервисами
   - Внешняя сеть Traefik при необходимости

Роль теперь более надежная и готова к использованию в production-средах.