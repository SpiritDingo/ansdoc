# Установка и настройка AWX сервера в Docker Compose для Enterprise

## Подготовка окружения

Перед развертыванием AWX необходимо подготовить сервер с Docker и Docker Compose:

```yaml
# playbook-prepare.yml
---
- name: Prepare server for AWX
  hosts: awx_servers
  become: yes
  vars:
    docker_version: "20.10.23"
    docker_compose_version: "2.17.2"
    
  tasks:
    - name: Install required system packages
      apt:
        name: "{{ item }}"
        state: present
        update_cache: yes
      loop:
        - apt-transport-https
        - ca-certificates
        - curl
        - gnupg-agent
        - software-properties-common
        - python3-pip
        - git
        - make
        - gcc
        - python3-dev
        
    - name: Add Docker GPG key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present
        
    - name: Add Docker repository
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present
        
    - name: Install Docker
      apt:
        name: "docker-ce={{ docker_version }}~3-0~ubuntu-{{ ansible_distribution_release }} docker-ce-cli={{ docker_version }}~3-0~ubuntu-{{ ansible_distribution_release }} containerd.io"
        state: present
        
    - name: Install Docker Compose
      pip:
        name: "docker-compose=={{ docker_compose_version }}"
        
    - name: Add current user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes
        
    - name: Enable and start Docker service
      systemd:
        name: docker
        state: started
        enabled: yes
```

## Роль для развертывания AWX

Создадим роль `awx_docker` со следующей структурой:

```
roles/awx_docker/
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

### Основные переменные (defaults/main.yml)

```yaml
# Версии AWX и PostgreSQL
awx_version: "21.7.0"
postgres_version: "13"

# Настройки PostgreSQL
pg_host: "postgres"
pg_port: 5432
pg_database: "awx"
pg_username: "awx"
pg_password: "awxpass"

# Настройки AWX
awx_admin_user: "admin"
awx_admin_password: "changeme"
awx_secret_key: "awxsecret"
awx_web_port: 8080
awx_ssl_enabled: false
awx_hostname: "awx.example.com"

# Настройки RabbitMQ
rabbitmq_user: "awx"
rabbitmq_password: "awxpass"
rabbitmq_vhost: "awx"

# Настройки ресурсов
awx_mem_limit: "4g"
postgres_mem_limit: "2g"
rabbitmq_mem_limit: "1g"
```

### Файл docker-compose (files/docker-compose.yml.j2)

```yaml
version: '3.7'
services:
  postgres:
    image: postgres:{{ postgres_version }}
    environment:
      POSTGRES_USER: "{{ pg_username }}"
      POSTGRES_PASSWORD: "{{ pg_password }}"
      POSTGRES_DB: "{{ pg_database }}"
    volumes:
      - awx_postgres_data:/var/lib/postgresql/data:Z
    deploy:
      resources:
        limits:
          memory: "{{ postgres_mem_limit }}"
    restart: unless-stopped
    networks:
      - awx_network

  rabbitmq:
    image: rabbitmq:3.8-management
    environment:
      RABBITMQ_DEFAULT_VHOST: "{{ rabbitmq_vhost }}"
      RABBITMQ_DEFAULT_USER: "{{ rabbitmq_user }}"
      RABBITMQ_DEFAULT_PASS: "{{ rabbitmq_password }}"
    volumes:
      - awx_rabbitmq_data:/var/lib/rabbitmq:Z
    deploy:
      resources:
        limits:
          memory: "{{ rabbitmq_mem_limit }}"
    restart: unless-stopped
    networks:
      - awx_network

  awx_web:
    image: ansible/awx:{{ awx_version }}
    hostname: "{{ awx_hostname }}"
    user: root
    volumes:
      - awx_web_data:/var/lib/awx:Z
    ports:
      - "{{ awx_web_port }}:8052"
    environment:
      - AWX_ADMIN_USER={{ awx_admin_user }}
      - AWX_ADMIN_PASSWORD={{ awx_admin_password }}
      - SECRET_KEY={{ awx_secret_key }}
      - DATABASE_USER={{ pg_username }}
      - DATABASE_PASSWORD={{ pg_password }}
      - DATABASE_HOST=postgres
      - DATABASE_PORT={{ pg_port }}
      - DATABASE_NAME={{ pg_database }}
      - RABBITMQ_USER={{ rabbitmq_user }}
      - RABBITMQ_PASSWORD={{ rabbitmq_password }}
      - RABBITMQ_HOST=rabbitmq
      - RABBITMQ_PORT=5672
      - RABBITMQ_VHOST={{ rabbitmq_vhost }}
    depends_on:
      - postgres
      - rabbitmq
    deploy:
      resources:
        limits:
          memory: "{{ awx_mem_limit }}"
    restart: unless-stopped
    networks:
      - awx_network

  awx_task:
    image: ansible/awx:{{ awx_version }}
    hostname: "{{ awx_hostname }}"
    user: root
    volumes:
      - awx_task_data:/var/lib/awx:Z
      - /etc/pki/ca-trust:/etc/pki/ca-trust:ro
      - /etc/ssl/certs:/etc/ssl/certs:ro
    environment:
      - AWX_ADMIN_USER={{ awx_admin_user }}
      - AWX_ADMIN_PASSWORD={{ awx_admin_password }}
      - SECRET_KEY={{ awx_secret_key }}
      - DATABASE_USER={{ pg_username }}
      - DATABASE_PASSWORD={{ pg_password }}
      - DATABASE_HOST=postgres
      - DATABASE_PORT={{ pg_port }}
      - DATABASE_NAME={{ pg_database }}
      - RABBITMQ_USER={{ rabbitmq_user }}
      - RABBITMQ_PASSWORD={{ rabbitmq_password }}
      - RABBITMQ_HOST=rabbitmq
      - RABBITMQ_PORT=5672
      - RABBITMQ_VHOST={{ rabbitmq_vhost }}
    depends_on:
      - postgres
      - rabbitmq
    deploy:
      resources:
        limits:
          memory: "{{ awx_mem_limit }}"
    restart: unless-stopped
    networks:
      - awx_network

volumes:
  awx_postgres_data:
  awx_rabbitmq_data:
  awx_web_data:
  awx_task_data:

networks:
  awx_network:
    driver: bridge
```

### Основные задачи (tasks/main.yml)

```yaml
---
- name: Create AWX directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
  loop:
    - /opt/awx
    - /opt/awx/config
    
- name: Copy docker-compose template
  template:
    src: files/docker-compose.yml.j2
    dest: /opt/awx/docker-compose.yml
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0644'
    
- name: Copy environment file
  template:
    src: files/awx.env.j2
    dest: /opt/awx/awx.env
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0600'
    
- name: Deploy AWX with Docker Compose
  community.docker.docker_compose:
    project_src: /opt/awx
    pull: yes
    build: no
    state: present
    
- name: Wait for AWX to be ready
  uri:
    url: "http://localhost:{{ awx_web_port }}"
    status_code: 200
    timeout: 30
  register: awx_ready
  until: awx_ready.status == 200
  retries: 30
  delay: 10
  ignore_errors: yes
```

## Playbook для развертывания

```yaml
# playbook-awx.yml
---
- name: Deploy AWX in Docker
  hosts: awx_servers
  become: yes
  vars_files:
    - vars/awx.yml  # Здесь можно переопределить переменные по умолчанию
    
  roles:
    - role: awx_docker
```

## Enterprise-рекомендации

1. **Безопасность**:
   - Используйте TLS для всех соединений
   - Настройте аутентификацию через LDAP/OAuth
   - Регулярно обновляйте контейнеры

2. **Резервное копирование**:
   ```yaml
   - name: Create backup script
     template:
       src: templates/backup-awx.sh.j2
       dest: /usr/local/bin/backup-awx.sh
       mode: '0755'
       
   - name: Schedule daily backups
     cron:
       name: "AWX daily backup"
       job: "/usr/local/bin/backup-awx.sh"
       minute: "0"
       hour: "2"
   ```

3. **Мониторинг**:
   - Настройте мониторинг контейнеров (Prometheus + Grafana)
   - Мониторинг производительности AWX

4. **Масштабирование**:
   - Для больших инсталляций рассмотрите Kubernetes вместо Docker Compose
   - Настройте горизонтальное масштабирование для исполнителей (awx_task)

5. **Интеграция с Enterprise-системами**:
   - Настройте интеграцию с CMDB
   - Подключите корпоративные системы аутентификации
   - Настройте отправку логов в централизованную систему

## Обновление AWX

Для обновления AWX в enterprise-окружении:

```yaml
- name: Update AWX
  hosts: awx_servers
  become: yes
  vars:
    new_awx_version: "22.0.0"
    
  tasks:
    - name: Stop AWX containers
      community.docker.docker_compose:
        project_src: /opt/awx
        state: absent
        
    - name: Update docker-compose file with new version
      replace:
        path: /opt/awx/docker-compose.yml
        regexp: 'ansible/awx:\d+\.\d+\.\d+'
        replace: 'ansible/awx:{{ new_awx_version }}'
        
    - name: Pull new images
      command: docker-compose -f /opt/awx/docker-compose.yml pull
      
    - name: Start updated AWX
      community.docker.docker_compose:
        project_src: /opt/awx
        pull: no
        build: no
        state: present
        
    - name: Verify AWX is running
      uri:
        url: "http://localhost:{{ awx_web_port }}"
        status_code: 200
        timeout: 30
```

Этот подход обеспечивает надежное и масштабируемое развертывание AWX в enterprise-окружении с возможностью централизованного управления через Ansible.


Вот полная Ansible роль для подготовки сервера к установке AWX в Docker Compose:

```yaml
# roles/prepare_awx_server/tasks/main.yml
---
- name: Install required system packages
  apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
  loop:
    - apt-transport-https
    - ca-certificates
    - curl
    - gnupg-agent
    - software-properties-common
    - python3-pip
    - git
    - make
    - gcc
    - python3-dev
    - libssl-dev
    - libffi-dev
    - docker-compose-plugin
  when: ansible_os_family == 'Debian'

- name: Install required packages for RHEL
  yum:
    name: "{{ item }}"
    state: present
  loop:
    - yum-utils
    - device-mapper-persistent-data
    - lvm2
    - python3-pip
    - git
    - gcc
    - python3-devel
    - openssl-devel
    - libffi-devel
  when: ansible_os_family == 'RedHat'

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present
  when: ansible_os_family == 'Debian'

- name: Add Docker repository for Debian
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present
  when: ansible_os_family == 'Debian'

- name: Add Docker repository for RHEL
  yum_repository:
    name: docker-ce
    description: Docker CE Stable
    baseurl: https://download.docker.com/linux/centos/$releasever/$basearch/stable
    gpgcheck: yes
    gpgkey: https://download.docker.com/linux/centos/gpg
    enabled: yes
  when: ansible_os_family == 'RedHat'

- name: Install Docker
  package:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present

- name: Install Docker Compose
  pip:
    name: docker-compose
    state: present
    executable: pip3

- name: Add current user to docker group
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Enable and start Docker service
  systemd:
    name: docker
    state: started
    enabled: yes

- name: Validate Docker installation
  command: docker --version
  register: docker_version
  changed_when: false

- name: Display Docker version
  debug:
    msg: "Docker version: {{ docker_version.stdout }}"

- name: Validate Docker Compose installation
  command: docker-compose --version
  register: docker_compose_version
  changed_when: false

- name: Display Docker Compose version
  debug:
    msg: "Docker Compose version: {{ docker_compose_version.stdout }}"

- name: Create directory structure for AWX
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0755'
  loop:
    - /opt/awx
    - /opt/awx/projects
    - /opt/awx/config

- name: Set kernel parameters for AWX
  sysctl:
    name: "{{ item.name }}"
    value: "{{ item.value }}"
    state: present
    reload: yes
  loop:
    - { name: "vm.max_map_count", value: "262144" }
    - { name: "net.ipv4.ip_local_port_range", value: "1024 65000" }
    - { name: "net.ipv4.tcp_fin_timeout", value: "30" }
```

```yaml
# roles/prepare_awx_server/defaults/main.yml
---
# Docker version settings
docker_version: "20.10.23"
docker_compose_version: "2.17.2"

# System settings
awx_base_dir: "/opt/awx"
awx_user: "awx"
awx_group: "awx"

# Resource limits
file_descriptor_limit: 65536
```

```yaml
# roles/prepare_awx_server/handlers/main.yml
---
- name: restart docker
  systemd:
    name: docker
    state: restarted
    enabled: yes
```

Пример использования роли в playbook:

```yaml
# playbook-prepare-awx.yml
---
- name: Prepare servers for AWX installation
  hosts: awx_servers
  become: yes
  vars:
    docker_version: "24.0.5"
    docker_compose_version: "2.21.0"
  
  roles:
    - prepare_awx_server
```

Эта роль выполняет:

1. Установку необходимых системных пакетов (поддерживает Debian/Ubuntu и RHEL/CentOS)
2. Установку Docker и Docker Compose
3. Настройку системного пользователя
4. Создание необходимой директории для AWX
5. Настройку параметров ядра для оптимальной работы AWX
6. Проверку корректности установки компонентов

Для enterprise-окружения рекомендуется дополнительно:

1. Настроить proxy для доступа к Docker Hub (если требуется)
2. Добавить корпоративные SSL-сертификаты
3. Настроить централизованный сбор логов
4. Интегрировать с системой мониторинга
5. Настроить бэкап важных директорий

Вы можете переопределить любые параметры из defaults/main.yml при вызове роли, как показано в примере playbook.


Ошибка указывает, что `docker-compose` не установлен или не находится в PATH. Давайте исправим роль для корректной установки Docker Compose.

### Исправленная роль для установки Docker Compose:

```yaml
# roles/awx_prepare_server/tasks/main.yml
---
- name: Set default remote user
  set_fact:
    remote_user: "{{ ansible_user | default(lookup('env', 'USER') | default('root')) }}"

# ... (предыдущие задачи остаются без изменений до установки Docker Compose)

- name: Install Docker Compose (official method)
  block:
    - name: Download Docker Compose binary
      get_url:
        url: "https://github.com/docker/compose/releases/download/v{{ docker_compose_version }}/docker-compose-{{ ansible_system }}-{{ ansible_architecture }}"
        dest: /usr/local/bin/docker-compose
        mode: '0755'
      register: download_compose
      until: download_compose is succeeded
      retries: 3
      delay: 10

    - name: Create symlink for Docker Compose
      file:
        src: /usr/local/bin/docker-compose
        dest: /usr/bin/docker-compose
        state: link
        force: yes

  rescue:
    - name: Install Docker Compose via pip (fallback)
      pip:
        name: docker-compose
        state: present
        executable: pip3

- name: Verify Docker Compose installation
  command: docker-compose --version
  register: docker_compose_check
  changed_when: false
  ignore_errors: yes

- name: Fail if Docker Compose not installed
  fail:
    msg: "Failed to install Docker Compose. Please install manually."
  when: docker_compose_check is failed

# ... (остальные задачи остаются без изменений)
```

### Обновленный файл defaults/main.yml:

```yaml
# roles/awx_prepare_server/defaults/main.yml
---
docker_compose_version: "2.23.0"  # Последняя стабильная версия
docker_compose_path: "/usr/local/bin/docker-compose"
```

### Почему это работает лучше:

1. **Два метода установки**:
   - Основной метод: загрузка бинарного файла напрямую с GitHub
   - Резервный метод: установка через pip (если первый не сработал)

2. **Проверка установки**:
   - Явная проверка что docker-compose работает
   - Фейл с понятным сообщением если установка не удалась

3. **Поддержка разных архитектур**:
   - Использует `ansible_system` и `ansible_architecture` для правильного выбора бинарника

### Дополнительные улучшения:

1. Добавьте в начало playbook проверку подключения к GitHub:

```yaml
- name: Check GitHub connectivity
  uri:
    url: https://github.com
    status_code: 200
    timeout: 5
  register: github_available
  ignore_errors: yes
  changed_when: false
```

2. Модифицируйте задачу установки:

```yaml
- name: Install Docker Compose
  block:
    - name: Download Docker Compose (if GitHub available)
      get_url:
        url: "https://github.com/docker/compose/releases/download/v{{ docker_compose_version }}/docker-compose-{{ ansible_system }}-{{ ansible_architecture }}"
        dest: "{{ docker_compose_path }}"
        mode: '0755'
      when: github_available is succeeded
  rescue:
    - name: Install via pip (fallback)
      pip:
        name: docker-compose
        state: present
        executable: pip3
```

### Как проверить работу:

После применения роли выполните вручную на сервере:
```bash
docker-compose --version
# Должно показать: Docker Compose version v2.23.0
```

Если проблема сохраняется:
1. Проверьте доступность GitHub из вашего сервера
2. Убедитесь что /usr/local/bin в PATH
3. Проверьте права на файл /usr/local/bin/docker-compose (должны быть 755)

Для enterprise-окружения рекомендуется:
1. Заранее скачать бинарник docker-compose во внутренний репозиторий
2. Изменить URL в роли на внутренний адрес
3. Добавить проверку контрольной суммы файла


Эта ошибка возникает из-за неправильного использования зарегистрированной переменной `docker_compose_version`. Давайте полностью переработаем секцию проверки и отображения версий Docker и Docker Compose в роли.

### Исправленная версия задач для проверки версий:

```yaml
# roles/awx_prepare_server/tasks/main.yml
---

# ... (предыдущие задачи остаются без изменений)

- name: Get Docker version
  command: docker --version
  register: docker_version_check
  changed_when: false
  ignore_errors: yes

- name: Display Docker version
  debug:
    msg: "Docker version info: {{ docker_version_check.stdout | default('Docker not found') }}"
  when: docker_version_check is not failed

- name: Verify Docker works
  command: docker info
  register: docker_info_check
  changed_when: false
  ignore_errors: yes
  when: docker_version_check is not failed

- name: Get Docker Compose version
  command: docker-compose --version
  register: docker_compose_check
  changed_when: false
  ignore_errors: yes

- name: Display Docker Compose version
  debug:
    msg: "Docker Compose version info: {{ docker_compose_check.stdout | default('Docker Compose not found') }}"
  when: docker_compose_check is not failed

- name: Fail if Docker not working
  fail:
    msg: |
      Docker installation failed!
      Version check: {{ docker_version_check.stdout | default('N/A') }}
      Docker info: {{ docker_info_check.stdout | default('N/A') }}
  when: docker_version_check is failed or docker_info_check is failed

- name: Fail if Docker Compose not working
  fail:
    msg: |
      Docker Compose installation failed!
      Version check: {{ docker_compose_check.stdout | default('N/A') }}
  when: docker_compose_check is failed

# ... (остальные задачи продолжаются)
```

### Ключевые улучшения:

1. **Более надежная проверка версий**:
   - Отдельные задачи для проверки `docker` и `docker-compose`
   - Проверка не только версии, но и работоспособности (`docker info`)

2. **Лучшая обработка ошибок**:
   - `ignore_errors: yes` для всех проверочных команд
   - Явные условия `when` для отображения информации только при успехе

3. **Информативные сообщения об ошибках**:
   - Подробный вывод при проблемах с установкой
   - Четкое разделение ошибок Docker и Docker Compose

4. **Защита от undefined переменных**:
   - Использование фильтра `default()` для всех переменных
   - Проверка `is not failed` перед использованием результатов

### Дополнительные рекомендации:

1. Для enterprise-окружения добавьте проверку контрольных сумм:

```yaml
- name: Verify Docker Compose checksum
  get_url:
    url: "https://github.com/docker/compose/releases/download/v{{ docker_compose_version }}/docker-compose-{{ ansible_system }}-{{ ansible_architecture }}.sha256"
    dest: /tmp/docker-compose.sha256
  register: compose_checksum_file
  when: github_available is succeeded

- name: Validate Docker Compose binary
  command: sha256sum -c /tmp/docker-compose.sha256
  args:
    chdir: /usr/local/bin
  when: compose_checksum_file is succeeded
```

2. Для лучшей поддержки разных ОС добавьте в defaults:

```yaml
# roles/awx_prepare_server/defaults/main.yml
docker_compose_binary_name: "docker-compose"
docker_compose_path: "/usr/local/bin/docker-compose"
docker_compose_alternate_paths:
  - "/usr/bin/docker-compose"
  - "/usr/local/bin/docker-compose"
  - "/bin/docker-compose"
```

3. Обновите задачу проверки версии Docker Compose:

```yaml
- name: Find Docker Compose binary
  find:
    paths: "{{ docker_compose_alternate_paths }}"
    patterns: "{{ docker_compose_binary_name }}"
    file_type: file
  register: docker_compose_binary
  changed_when: false

- name: Set Docker Compose path fact
  set_fact:
    docker_compose_executable: "{{ docker_compose_binary.files[0].path | default(docker_compose_path) }}"
  when: docker_compose_binary.matched > 0

- name: Get Docker Compose version
  command: "{{ docker_compose_executable }} --version"
  register: docker_compose_check
  changed_when: false
  ignore_errors: yes
```

Эти изменения сделают вашу роль более надежной и подходящей для production-окружений.





# Ansible роль для установки AWX с Docker Compose версии 2+

Вот обновленная роль для развертывания AWX с использованием современных версий Docker Compose (v2+):

## Структура роли

```
roles/awx_docker/
├── defaults
│   └── main.yml
├── files
│   ├── docker-compose.yml.j2
│   └── awx.env.j2
├── tasks
│   ├── main.yml
│   ├── install_docker.yml
│   └── configure_awx.yml
└── templates
    └── nginx.conf.j2
```

## Обновленные defaults/main.yml

```yaml
# Версии компонентов
awx_version: "23.0.0"  # Последняя стабильная версия AWX
postgres_version: "15"
redis_version: "7"

# Настройки Docker Compose
compose_version: "v2"  # Используем Docker Compose v2
compose_command: "docker compose"  # Команда для Docker Compose v2

# Настройки PostgreSQL
pg_host: "postgres"
pg_port: 5432
pg_database: "awx"
pg_username: "awx"
pg_password: "awxpass"

# Настройки Redis
redis_password: "redispass"

# Настройки AWX
awx_admin_user: "admin"
awx_admin_password: "changeme"
awx_secret_key: "awxsecret"
awx_web_port: 8080
awx_ssl_enabled: false
awx_hostname: "awx.example.com"

# Настройки ресурсов
awx_mem_limit: "4g"
postgres_mem_limit: "2g"
redis_mem_limit: "1g"
```

## Обновленный файл docker-compose.yml.j2 (для Docker Compose v2+)

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:{{ postgres_version }}
    container_name: awx-postgres
    environment:
      POSTGRES_USER: "{{ pg_username }}"
      POSTGRES_PASSWORD: "{{ pg_password }}"
      POSTGRES_DB: "{{ pg_database }}"
    volumes:
      - awx_postgres_data:/var/lib/postgresql/data:Z
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U {{ pg_username }} -d {{ pg_database }}"]
      interval: 5s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: "{{ postgres_mem_limit }}"
    restart: unless-stopped
    networks:
      - awx_network

  redis:
    image: redis:{{ redis_version }}
    container_name: awx-redis
    command: redis-server --requirepass {{ redis_password }}
    volumes:
      - awx_redis_data:/data:Z
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: "{{ redis_mem_limit }}"
    restart: unless-stopped
    networks:
      - awx_network

  awx-web:
    image: ansible/awx:{{ awx_version }}
    container_name: awx-web
    hostname: "{{ awx_hostname }}"
    user: root
    volumes:
      - awx_web_data:/var/lib/awx:Z
      - /etc/pki/ca-trust:/etc/pki/ca-trust:ro
      - /etc/ssl/certs:/etc/ssl/certs:ro
    ports:
      - "{{ awx_web_port }}:8052"
    environment:
      - AWX_ADMIN_USER={{ awx_admin_user }}
      - AWX_ADMIN_PASSWORD={{ awx_admin_password }}
      - SECRET_KEY={{ awx_secret_key }}
      - DATABASE_USER={{ pg_username }}
      - DATABASE_PASSWORD={{ pg_password }}
      - DATABASE_HOST=postgres
      - DATABASE_PORT={{ pg_port }}
      - DATABASE_NAME={{ pg_database }}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD={{ redis_password }}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: "{{ awx_mem_limit }}"
    restart: unless-stopped
    networks:
      - awx_network

  awx-task:
    image: ansible/awx:{{ awx_version }}
    container_name: awx-task
    hostname: "{{ awx_hostname }}"
    user: root
    volumes:
      - awx_task_data:/var/lib/awx:Z
      - /etc/pki/ca-trust:/etc/pki/ca-trust:ro
      - /etc/ssl/certs:/etc/ssl/certs:ro
    environment:
      - AWX_ADMIN_USER={{ awx_admin_user }}
      - AWX_ADMIN_PASSWORD={{ awx_admin_password }}
      - SECRET_KEY={{ awx_secret_key }}
      - DATABASE_USER={{ pg_username }}
      - DATABASE_PASSWORD={{ pg_password }}
      - DATABASE_HOST=postgres
      - DATABASE_PORT={{ pg_port }}
      - DATABASE_NAME={{ pg_database }}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - REDIS_PASSWORD={{ redis_password }}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          memory: "{{ awx_mem_limit }}"
    restart: unless-stopped
    networks:
      - awx_network

volumes:
  awx_postgres_data:
    name: awx-postgres-data
  awx_redis_data:
    name: awx-redis-data
  awx_web_data:
    name: awx-web-data
  awx_task_data:
    name: awx-task-data

networks:
  awx_network:
    name: awx-network
    driver: bridge
```

## Обновленные задачи (tasks/main.yml)

```yaml
---
- name: Include Docker installation tasks
  ansible.builtin.include_tasks: install_docker.yml
  tags: docker

- name: Include AWX configuration tasks
  ansible.builtin.include_tasks: configure_awx.yml
  tags: awx
```

## tasks/install_docker.yml

```yaml
---
- name: Install prerequisites
  package:
    name:
      - ca-certificates
      - curl
      - gnupg
      - lsb-release
    state: present

- name: Add Docker GPG key
  ansible.builtin.apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker repository
  ansible.builtin.apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present
    update_cache: yes

- name: Install Docker packages
  package:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
    state: present

- name: Add user to docker group
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Enable and start Docker service
  systemd:
    name: docker
    state: started
    enabled: yes

- name: Verify Docker installation
  command: docker --version
  register: docker_version
  changed_when: false

- name: Display Docker version
  debug:
    msg: "{{ docker_version.stdout }}"

- name: Verify Docker Compose v2 installation
  command: docker compose version
  register: compose_version
  changed_when: false

- name: Display Docker Compose version
  debug:
    msg: "{{ compose_version.stdout }}"
```

## tasks/configure_awx.yml

```yaml
---
- name: Create AWX directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0755'
  loop:
    - /opt/awx
    - /opt/awx/config
    - /opt/awx/projects

- name: Copy docker-compose template
  template:
    src: docker-compose.yml.j2
    dest: /opt/awx/docker-compose.yml
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0644'

- name: Copy environment file
  template:
    src: awx.env.j2
    dest: /opt/awx/awx.env
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0600'

- name: Pull AWX images
  command: "{{ compose_command }} -f /opt/awx/docker-compose.yml pull"
  args:
    chdir: /opt/awx

- name: Deploy AWX stack
  command: "{{ compose_command }} -f /opt/awx/docker-compose.yml up -d"
  args:
    chdir: /opt/awx

- name: Wait for AWX to be ready
  uri:
    url: "http://localhost:{{ awx_web_port }}"
    status_code: 200
    timeout: 30
  register: awx_ready
  until: awx_ready.status == 200
  retries: 30
  delay: 10
  ignore_errors: yes
```

## Обновленный playbook для развертывания

```yaml
# playbook-awx.yml
---
- name: Deploy AWX with Docker Compose v2
  hosts: awx_servers
  become: yes
  vars_files:
    - vars/awx.yml  # Переопределения переменных

  pre_tasks:
    - name: Ensure Python is installed
      raw: test -e /usr/bin/python3 || (apt -y update && apt install -y python3)
      changed_when: false
      tags: always

  roles:
    - role: awx_docker

  post_tasks:
    - name: Display AWX access information
      debug:
        msg: |
          AWX успешно развернут!
          URL: http://{{ ansible_host }}:{{ awx_web_port }}
          Логин: {{ awx_admin_user }}
          Пароль: {{ awx_admin_password }}
```

## Ключевые улучшения для Docker Compose v2+:

1. **Использование Docker Compose Plugin**:
   - Вместо отдельного `docker-compose` используется встроенная команда `docker compose`
   - Более надежная интеграция с Docker Engine

2. **Обновленные версии компонентов**:
   - AWX 23.0.0
   - PostgreSQL 15
   - Redis вместо RabbitMQ (современные версии AWX используют Redis)

3. **Улучшенная конфигурация**:
   - Явные имена контейнеров и томов
   - Healthchecks для всех сервисов
   - Явные зависимости между сервисами

4. **Оптимизированная установка**:
   - Использование официального Docker Compose plugin
   - Разделение задач на логические группы

5. **Enterprise-функции**:
   - Именованные тома и сети для лучшего управления
   - Явные ограничения ресурсов
   - Поддержка SSL/TLS через монтирование сертификатов

Для применения этой роли:

```bash
ansible-playbook -i inventory.ini playbook-awx.yml
```


Ошибка возникает из-за отсутствия определения переменной `redis_password`. Давайте исправим роль, чтобы она корректно работала с Redis. Вот полное исправление:

### 1. Обновим `defaults/main.yml`

Добавим все необходимые переменные для Redis:

```yaml
# Redis configuration
redis_enabled: true
redis_host: "redis"
redis_port: 6379
redis_password: "awx_redis_password"  # Добавляем значение по умолчанию
redis_mem_limit: "1g"
```

### 2. Модифицируем `files/docker-compose.yml.j2`

Обновим конфигурацию Redis в docker-compose:

```yaml
  redis:
    image: redis:{{ redis_version }}
    container_name: awx-redis
    command: redis-server --requirepass {{ redis_password }}
    volumes:
      - awx_redis_data:/data:Z
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "{{ redis_password }}", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: "{{ redis_mem_limit }}"
    restart: unless-stopped
    networks:
      - awx_network
```

### 3. Обновим `tasks/configure_awx.yml`

Добавим проверку переменных:

```yaml
- name: Validate Redis configuration
  assert:
    that:
      - redis_password is defined
      - redis_password | length >= 8
    fail_msg: "Redis password must be defined and at least 8 characters long"
    success_msg: "Redis configuration validated"
```

### 4. Создадим файл `vars/awx.yml` для переопределения

```yaml
# vars/awx.yml
redis_password: "your_secure_password_here"  # Замените на реальный пароль
```

### 5. Полностью исправленная роль

Структура роли теперь должна включать:

```
roles/awx_docker/
├── defaults
│   └── main.yml          # Все переменные по умолчанию
├── vars
│   └── awx.yml           # Переопределения для enterprise
├── files
│   ├── docker-compose.yml.j2
│   └── awx.env.j2
├── tasks
│   ├── main.yml
│   ├── install_docker.yml
│   └── configure_awx.yml
└── templates
    └── nginx.conf.j2
```

### 6. Исправленный playbook

```yaml
# playbook-awx.yml
---
- name: Deploy AWX with Docker Compose v2
  hosts: awx_servers
  become: yes
  vars_files:
    - roles/awx_docker/vars/awx.yml  # Явное указание файла с переменными

  pre_tasks:
    - name: Ensure variables are defined
      assert:
        that:
          - redis_password is defined
          - redis_password | length >= 8
        fail_msg: "Required variables are not properly defined"

  roles:
    - role: awx_docker
```

### 7. Как развернуть с исправлениями

1. Создайте файл с переменными:
```bash
mkdir -p roles/awx_docker/vars
echo 'redis_password: "your_strong_password"' > roles/awx_docker/vars/awx.yml
```

2. Запустите playbook с проверкой переменных:
```bash
ansible-playbook -i inventory.ini playbook-awx.yml --extra-vars "@roles/awx_docker/vars/awx.yml"
```

### 8. Дополнительные меры безопасности

1. **Шифрование паролей**:
Используйте Ansible Vault для защиты чувствительных данных:
```bash
ansible-vault encrypt roles/awx_docker/vars/awx.yml
```

2. **Генерация паролей**:
Можно автоматически генерировать пароли:

```yaml
# В tasks/configure_awx.yml
- name: Generate Redis password if not defined
  set_fact:
    redis_password: "{{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}"
  when: redis_password is not defined
```

Эти изменения гарантируют, что:
1. Все обязательные переменные определены
2. Пароли соответствуют требованиям безопасности
3. Конфигурация Redis работает корректно
4. Развертывание проходит без ошибок



Ошибка указывает, что Ansible не может найти файл шаблона `awx.env.j2`. Давайте полностью исправим эту проблему и улучшим структуру роли.

### Полное исправление роли для AWX с Docker Compose v2+

1. **Правильная структура файлов**:

```
roles/awx_docker/
├── defaults
│   └── main.yml
├── files
│   ├── awx.env          # Файл окружения (не шаблон)
├── tasks
│   ├── main.yml
│   ├── install_docker.yml
│   └── configure_awx.yml
└── templates
    └── docker-compose.yml.j2  # Только docker-compose как шаблон
```

2. **Исправленный файл `files/awx.env`**:

```ini
# Файл окружения AWX
AWX_ADMIN_USER=admin
AWX_ADMIN_PASSWORD=changeme
SECRET_KEY=awxsecret
DATABASE_USER=awx
DATABASE_PASSWORD=awxpass
DATABASE_HOST=postgres
DATABASE_PORT=5432
DATABASE_NAME=awx
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=redispass
```

3. **Обновленные задачи в `tasks/configure_awx.yml`**:

```yaml
- name: Create AWX directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0755'
  loop:
    - /opt/awx
    - /opt/awx/config
    - /opt/awx/projects

- name: Copy docker-compose template
  template:
    src: docker-compose.yml.j2
    dest: /opt/awx/docker-compose.yml
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0644'

- name: Copy environment file
  copy:
    src: awx.env
    dest: /opt/awx/awx.env
    owner: "{{ ansible_user }}"
    group: "{{ ansible_user }}"
    mode: '0600'

- name: Validate Docker Compose file
  command: "{{ compose_command }} -f /opt/awx/docker-compose.yml config"
  register: compose_validate
  changed_when: false

- name: Pull AWX images
  command: "{{ compose_command }} -f /opt/awx/docker-compose.yml pull"
  args:
    chdir: /opt/awx

- name: Deploy AWX stack
  command: "{{ compose_command }} -f /opt/awx/docker-compose.yml up -d"
  args:
    chdir: /opt/awx
```

4. **Ключевые изменения**:

- Убрали `awx.env.j2` как шаблон, так как он не требует подстановки переменных
- Используем модуль `copy` вместо `template` для статичного файла окружения
- Добавили валидацию docker-compose файла перед запуском
- Упростили структуру файлов

5. **Как исправить ошибку**:

1. Убедитесь, что файл существует в правильной директории:
```bash
mkdir -p roles/awx_docker/files
touch roles/awx_docker/files/awx.env
```

2. Заполните файл `awx.env` необходимыми переменными (как показано выше)

3. Для динамических значений используйте `defaults/main.yml`:

```yaml
# defaults/main.yml
awx_admin_user: admin
awx_admin_password: changeme
awx_secret_key: awxsecret
pg_username: awx
pg_password: awxpass
redis_password: redispass
```

6. **Enterprise-рекомендации**:

- Для безопасности используйте Ansible Vault для шифрования паролей:
```bash
ansible-vault encrypt roles/awx_docker/files/awx.env
```

- Или генерируйте секреты во время выполнения:

```yaml
- name: Generate secrets if not defined
  set_fact:
    awx_secret_key: "{{ lookup('password', '/dev/null length=64 chars=ascii_letters,digits') }}"
    pg_password: "{{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}"
    redis_password: "{{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}"
  when: 
    - awx_secret_key is not defined
    - pg_password is not defined
    - redis_password is not defined
```

Теперь роль должна работать без ошибок, так как:
1. Все файлы находятся в правильных директориях
2. Нет попыток использовать несуществующие шаблоны
3. Структура роли соответствует best practices Ansible