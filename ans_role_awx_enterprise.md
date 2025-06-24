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