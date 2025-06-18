# Ansible роль для установки и настройки Docker Compose с интеграцией AWX, NetBox, Terraform и динамическим инвентарём

## Описание роли

Эта роль развертывает стек инструментов DevOps в Docker Compose, включая:
- Nginx как reverse proxy
- AWX (Ansible Tower的开源版本)
- NetBox (IPAM и DCIM система)
- Ansible
- Terraform

Роль также настраивает интеграцию между этими инструментами, включая динамический инвентарь из NetBox в AWX и Ansible.

## Структура роли

```
awx_netbox_stack/
├── defaults/
│   └── main.yml       # Переменные по умолчанию
├── files/
│   ├── nginx/         # Конфиги Nginx
│   ├── awx/           # Конфиги AWX
│   ├── netbox/        # Конфиги NetBox
│   └── scripts/       # Вспомогательные скрипты
├── tasks/
│   ├── main.yml       # Основные задачи
│   ├── docker.yml     # Установка Docker
│   ├── compose.yml    # Развертывание Compose
│   ├── nginx.yml      # Настройка Nginx
│   ├── awx.yml        # Настройка AWX
│   ├── netbox.yml     # Настройка NetBox
│   ├── integration.yml # Интеграция компонентов
│   └── verification.yml # Проверка работы
├── templates/
│   ├── docker-compose.j2 # Шаблон docker-compose
│   └── nginx.conf.j2    # Шаблон конфига Nginx
└── meta/main.yml      # Зависимости роли
```

## Основные задачи (tasks/main.yml)

```yaml
---
- name: Include pre-installation tasks
  include_tasks: pre_install.yml

- name: Install Docker and Docker Compose
  include_tasks: docker.yml

- name: Deploy Docker Compose stack
  include_tasks: compose.yml

- name: Configure Nginx as reverse proxy
  include_tasks: nginx.yml

- name: Configure AWX
  include_tasks: awx.yml

- name: Configure NetBox
  include_tasks: netbox.yml

- name: Set up integration between components
  include_tasks: integration.yml

- name: Verify installation
  include_tasks: verification.yml
```

## Шаблон docker-compose (templates/docker-compose.j2)

```yaml
version: '3.7'

services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/certs:/etc/nginx/certs
    depends_on:
      - awx
      - netbox
    networks:
      - netbox_awx_network

  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: awx
      POSTGRES_USER: awx
      POSTGRES_PASSWORD: "{{ awx_postgres_password }}"
    volumes:
      - awx_postgres_data:/var/lib/postgresql/data
    networks:
      - netbox_awx_network

  redis:
    image: redis:latest
    networks:
      - netbox_awx_network

  awx:
    image: ansible/awx:latest
    hostname: awx
    environment:
      AWX_ADMIN_USER: admin
      AWX_ADMIN_PASSWORD: "{{ awx_admin_password }}"
      DATABASE_HOST: postgres
      DATABASE_PORT: 5432
      DATABASE_NAME: awx
      DATABASE_USER: awx
      DATABASE_PASSWORD: "{{ awx_postgres_password }}"
      RABBITMQ_USER: guest
      RABBITMQ_PASS: guest
      MEMCACHED_HOST: memcached
      RABBITMQ_HOST: rabbitmq
      AWX_SKIP_MIGRATIONS: "false"
    volumes:
      - awx_projects:/var/lib/awx/projects
    depends_on:
      - postgres
      - redis
    networks:
      - netbox_awx_network

  netbox:
    image: netboxcommunity/netbox:latest
    depends_on:
      - netbox-postgres
      - netbox-redis
    environment:
      - DB_NAME=netbox
      - DB_USER=netbox
      - DB_PASSWORD={{ netbox_postgres_password }}
      - DB_HOST=netbox-postgres
      - DB_PORT=5432
      - REDIS_HOST=netbox-redis
      - REDIS_PORT=6379
      - SUPERUSER_NAME=admin
      - SUPERUSER_EMAIL=admin@example.com
      - SUPERUSER_PASSWORD={{ netbox_admin_password }}
      - ALLOWED_HOSTS=*
    volumes:
      - netbox-media-files:/opt/netbox/netbox/media
    networks:
      - netbox_awx_network

  netbox-postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=netbox
      - POSTGRES_USER=netbox
      - POSTGRES_PASSWORD={{ netbox_postgres_password }}
    volumes:
      - netbox-postgres-data:/var/lib/postgresql/data
    networks:
      - netbox_awx_network

  netbox-redis:
    image: redis:6
    networks:
      - netbox_awx_network

volumes:
  awx_postgres_data:
  awx_projects:
  netbox-postgres-data:
  netbox-media-files:

networks:
  netbox_awx_network:
    driver: bridge
```

## Настройка динамического инвентаря (tasks/integration.yml)

```yaml
---
- name: Install NetBox inventory script for Ansible
  get_url:
    url: https://raw.githubusercontent.com/netbox-community/ansible_modules/develop/netbox_inventory.py
    dest: /usr/local/bin/netbox_inventory.py
    mode: '0755'

- name: Create NetBox inventory config
  template:
    src: netbox_inventory.cfg.j2
    dest: /etc/ansible/netbox_inventory.cfg
    owner: root
    group: root
    mode: '0644'

- name: Configure AWX to use NetBox as dynamic inventory source
  uri:
    url: "http://awx/api/v2/inventory_scripts/"
    method: POST
    body_format: json
    body:
      name: "NetBox Dynamic Inventory"
      script: "{{ lookup('file', '/usr/local/bin/netbox_inventory.py') }}"
      organization: 1
    status_code: 201
    user: "{{ awx_admin_user }}"
    password: "{{ awx_admin_password }}"
    force_basic_auth: yes
    headers:
      Content-Type: "application/json"
  register: awx_inventory_script
  when: awx_inventory_script is not defined

- name: Create inventory in AWX
  uri:
    url: "http://awx/api/v2/inventories/"
    method: POST
    body_format: json
    body:
      name: "NetBox Inventory"
      organization: 1
      variables: |
        {
          "netbox_inventory_config": "/etc/ansible/netbox_inventory.cfg"
        }
    status_code: 201
    user: "{{ awx_admin_user }}"
    password: "{{ awx_admin_password }}"
    force_basic_auth: yes
    headers:
      Content-Type: "application/json"
  register: awx_inventory
  when: awx_inventory is not defined

- name: Add inventory source to AWX inventory
  uri:
    url: "http://awx/api/v2/inventories/{{ awx_inventory.json.id }}/inventory_sources/"
    method: POST
    body_format: json
    body:
      name: "NetBox Source"
      source: "custom"
      source_script: "{{ awx_inventory_script.json.id }}"
      source_vars: |
        {
          "group_by": "device_role",
          "filter": {
            "status": "active"
          }
        }
    status_code: 201
    user: "{{ awx_admin_user }}"
    password: "{{ awx_admin_password }}"
    force_basic_auth: yes
    headers:
      Content-Type: "application/json"
  when: awx_inventory is defined and awx_inventory_script is defined
```

## Настройка интеграции с Terraform (tasks/integration.yml продолжение)

```yaml
- name: Install Terraform in AWX execution environment
  docker_exec:
    container: awx
    command: |
      apt-get update && \
      apt-get install -y wget unzip && \
      wget https://releases.hashicorp.com/terraform/{{ terraform_version }}/terraform_{{ terraform_version }}_linux_amd64.zip && \
      unzip terraform_{{ terraform_version }}_linux_amd64.zip -d /usr/local/bin && \
      rm terraform_{{ terraform_version }}_linux_amd64.zip

- name: Create Terraform project directory in AWX
  file:
    path: /var/lib/awx/projects/terraform
    state: directory
    mode: '0755'

- name: Configure AWX to run Terraform playbooks
  uri:
    url: "http://awx/api/v2/job_templates/"
    method: POST
    body_format: json
    body:
      name: "Terraform Apply"
      job_type: "run"
      inventory: "{{ awx_inventory.json.id }}"
      project: 1
      playbook: "terraform_apply.yml"
      credential: 1
      extra_vars: |
        {
          "tf_dir": "{{ terraform_project_dir }}"
        }
    status_code: 201
    user: "{{ awx_admin_user }}"
    password: "{{ awx_admin_password }}"
    force_basic_auth: yes
    headers:
      Content-Type: "application/json"
  when: awx_inventory is defined
```

## Пример плейбука для запуска Terraform через AWX (files/terraform_apply.yml)

```yaml
---
- name: Run Terraform
  hosts: localhost
  gather_facts: no
  tasks:
    - name: Initialize Terraform
      command: terraform init
      args:
        chdir: "{{ tf_dir }}"
      register: init_result

    - name: Apply Terraform
      command: terraform apply -auto-approve
      args:
        chdir: "{{ tf_dir }}"
      register: apply_result
      when: init_result.rc == 0

    - name: Show Terraform output
      debug:
        var: apply_result.stdout
      when: apply_result is defined
```

## Переменные по умолчанию (defaults/main.yml)

```yaml
---
# AWX settings
awx_admin_user: admin
awx_admin_password: "ChangeMe123!"
awx_postgres_password: "AWXpgPass123"

# NetBox settings
netbox_admin_password: "NetBoxAdmin123!"
netbox_postgres_password: "NetBoxpgPass123"

# Terraform settings
terraform_version: "1.2.3"
terraform_project_dir: "/var/lib/awx/projects/terraform"

# Network settings
awx_hostname: awx.example.com
netbox_hostname: netbox.example.com
```

## Использование роли

1. Создайте playbook для развертывания:

```yaml
---
- hosts: localhost
  become: yes
  roles:
    - awx_netbox_stack
```

2. Запустите playbook:

```bash
ansible-playbook -i localhost, deploy_stack.yml
```

После развертывания вы получите:
- AWX доступный на http://awx.example.com
- NetBox доступный на http://netbox.example.com
- Настроенный динамический инвентарь из NetBox в AWX
- Возможность запускать Terraform через AWX

Для работы интеграции необходимо дополнительно настроить:
1. API токены в NetBox
2. Учетные данные в AWX
3. Terraform проекты в соответствующей директории





# Полная Ansible роль для развертывания Docker Compose стека с AWX, NetBox, Nginx и интеграцией

## Структура роли

```
awx_netbox_stack/
├── defaults/
│   └── main.yml
├── files/
│   ├── awx/
│   │   └── settings.py
│   ├── nginx/
│   │   ├── nginx.conf
│   │   └── certs/
│   ├── netbox/
│   │   ├── configuration.py
│   │   └── ldap_config.py (если нужно)
│   └── scripts/
│       ├── netbox_inventory.py
│       └── terraform_wrapper.sh
├── handlers/
│   └── main.yml
├── meta/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── pre_install.yml
│   ├── docker.yml
│   ├── compose.yml
│   ├── nginx.yml
│   ├── awx.yml
│   ├── netbox.yml
│   ├── integration.yml
│   └── verification.yml
├── templates/
│   ├── docker-compose.j2
│   ├── nginx.conf.j2
│   ├── netbox_inventory.cfg.j2
│   └── awx_terraform_playbook.j2
└── vars/
    └── main.yml
```

## 1. defaults/main.yml

```yaml
---
# Общие настройки
stack_dir: "/opt/netbox_awx_stack"
backup_dir: "/var/backups/netbox_awx"

# Docker настройки
docker_compose_version: "1.29.2"

# Nginx настройки
nginx_http_port: 80
nginx_https_port: 443
ssl_cert_path: "{{ stack_dir }}/nginx/certs/fullchain.pem"
ssl_key_path: "{{ stack_dir }}/nginx/certs/privkey.pem"

# AWX настройки
awx_version: "latest"
awx_admin_user: "admin"
awx_admin_password: "ChangeMeAwx123!"
awx_secret_key: "awxsecretkeychangeme"
awx_pg_db: "awx"
awx_pg_user: "awx"
awx_pg_password: "AwxPgPass123"
awx_pg_port: 5432
awx_redis_port: 6379
awx_hostname: "awx.example.com"
awx_project_dir: "{{ stack_dir }}/awx/projects"

# NetBox настройки
netbox_version: "latest"
netbox_secret_key: "netboxsecretkeychangeme"
netbox_admin_user: "admin"
netbox_admin_email: "admin@example.com"
netbox_admin_password: "ChangeMeNetbox123!"
netbox_pg_db: "netbox"
netbox_pg_user: "netbox"
netbox_pg_password: "NetBoxPgPass123"
netbox_pg_port: 5432
netbox_redis_port: 6379
netbox_hostname: "netbox.example.com"

# Настройки интеграции
netbox_api_token: ""
netbox_inventory_config: "/etc/ansible/netbox_inventory.cfg"
netbox_inventory_group_by: "device_role"
netbox_inventory_filter: "{'status': 'active'}"

# Terraform настройки
terraform_version: "1.5.7"
terraform_plugins_dir: "/usr/local/share/terraform/plugins"
```

## 2. vars/main.yml

```yaml
---
# Приватные переменные (можно зашифровать с помощью ansible-vault)
# Эти переменные переопределяют defaults
```

## 3. handlers/main.yml

```yaml
---
- name: restart docker
  systemd:
    name: docker
    state: restarted
    enabled: yes

- name: restart nginx
  systemd:
    name: nginx
    state: restarted
    enabled: yes

- name: reload docker compose
  command: docker-compose -f {{ stack_dir }}/docker-compose.yml down && docker-compose -f {{ stack_dir }}/docker-compose.yml up -d
  args:
    chdir: "{{ stack_dir }}"
```

## 4. tasks/main.yml

```yaml
---
- name: Include pre-installation tasks
  include_tasks: pre_install.yml
  tags: pre_install

- name: Install Docker and dependencies
  include_tasks: docker.yml
  tags: docker

- name: Deploy Docker Compose stack
  include_tasks: compose.yml
  tags: compose

- name: Configure Nginx reverse proxy
  include_tasks: nginx.yml
  tags: nginx

- name: Configure AWX
  include_tasks: awx.yml
  tags: awx

- name: Configure NetBox
  include_tasks: netbox.yml
  tags: netbox

- name: Set up integration between components
  include_tasks: integration.yml
  tags: integration

- name: Verify installation
  include_tasks: verification.yml
  tags: verification
```

## 5. tasks/pre_install.yml

```yaml
---
- name: Check if running on supported OS
  fail:
    msg: "This role requires Ubuntu 20.04/22.04 or CentOS 7/8"
  when: >
    ansible_distribution not in ['Ubuntu', 'CentOS'] or
    (ansible_distribution == 'Ubuntu' and ansible_distribution_version not in ['20.04', '22.04']) or
    (ansible_distribution == 'CentOS' and ansible_distribution_major_version not in ['7', '8'])

- name: Create stack directory
  file:
    path: "{{ stack_dir }}"
    state: directory
    mode: '0755'

- name: Create backup directory
  file:
    path: "{{ backup_dir }}"
    state: directory
    mode: '0755'

- name: Install prerequisite packages
  apt:
    name:
      - python3-pip
      - git
      - curl
      - wget
      - unzip
      - make
      - gcc
      - libssl-dev
      - libffi-dev
    state: present
    update_cache: yes
  when: ansible_distribution == 'Ubuntu'

- name: Install prerequisite packages (CentOS)
  yum:
    name:
      - python3-pip
      - git
      - curl
      - wget
      - unzip
      - make
      - gcc
      - openssl-devel
      - libffi-devel
    state: present
  when: ansible_distribution == 'CentOS'
```

## 6. tasks/docker.yml

```yaml
---
- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/{{ ansible_distribution | lower }}/gpg
    state: present
  when: ansible_distribution == 'Ubuntu'

- name: Add Docker repository (Ubuntu)
  apt_repository:
    repo: "deb [arch=amd64] https://download.docker.com/linux/{{ ansible_distribution | lower }} {{ ansible_distribution_release }} stable"
    state: present
    update_cache: yes
  when: ansible_distribution == 'Ubuntu'

- name: Add Docker repository (CentOS)
  yum_repository:
    name: docker-ce
    description: Docker CE Stable
    baseurl: https://download.docker.com/linux/centos/$releasever/$basearch/stable
    gpgcheck: yes
    gpgkey: https://download.docker.com/linux/centos/gpg
    enabled: yes
  when: ansible_distribution == 'CentOS'

- name: Install Docker
  package:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present

- name: Install Docker Compose
  get_url:
    url: "https://github.com/docker/compose/releases/download/{{ docker_compose_version }}/docker-compose-Linux-x86_64"
    dest: "/usr/local/bin/docker-compose"
    mode: '0755'

- name: Ensure Docker is running and enabled
  systemd:
    name: docker
    state: started
    enabled: yes

- name: Add current user to docker group
  user:
    name: "{{ ansible_user_id }}"
    groups: docker
    append: yes
```

## 7. tasks/compose.yml

```yaml
---
- name: Create directory structure
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ stack_dir }}/awx"
    - "{{ stack_dir }}/netbox"
    - "{{ stack_dir }}/nginx"
    - "{{ stack_dir }}/nginx/certs"
    - "{{ awx_project_dir }}"

- name: Deploy docker-compose template
  template:
    src: docker-compose.j2
    dest: "{{ stack_dir }}/docker-compose.yml"
    mode: '0644'

- name: Pull Docker images
  command: docker-compose -f {{ stack_dir }}/docker-compose.yml pull
  args:
    chdir: "{{ stack_dir }}"

- name: Start containers
  command: docker-compose -f {{ stack_dir }}/docker-compose.yml up -d
  args:
    chdir: "{{ stack_dir }}"
```

## 8. tasks/nginx.yml

```yaml
---
- name: Install Nginx
  package:
    name: nginx
    state: present

- name: Deploy Nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: '0644'
  notify: restart nginx

- name: Ensure Nginx is running
  systemd:
    name: nginx
    state: started
    enabled: yes

- name: Generate self-signed SSL certificate (if no cert provided)
  command: |
    openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout {{ ssl_key_path }} \
    -out {{ ssl_cert_path }} \
    -subj "/CN={{ netbox_hostname }}"
  args:
    creates: "{{ ssl_cert_path }}"
  when: not lookup('file', ssl_cert_path, errors='ignore')
```

## 9. tasks/awx.yml

```yaml
---
- name: Wait for AWX to be ready
  uri:
    url: "http://localhost:8052"
    status_code: 200
    timeout: 30
  register: awx_ready
  until: awx_ready.status == 200
  retries: 30
  delay: 10
  ignore_errors: yes

- name: Create AWX admin user via API
  uri:
    url: "http://localhost:8052/api/v2/users/"
    method: POST
    body_format: json
    body:
      username: "{{ awx_admin_user }}"
      password: "{{ awx_admin_password }}"
      email: "admin@example.com"
      is_superuser: true
      is_staff: true
    status_code: 201
    headers:
      Content-Type: "application/json"
  when: awx_ready.status == 200

- name: Create default organization in AWX
  uri:
    url: "http://localhost:8052/api/v2/organizations/"
    method: POST
    body_format: json
    body:
      name: "Default"
      description: "Default Organization"
    status_code: 201
    headers:
      Content-Type: "application/json"
  when: awx_ready.status == 200
```

## 10. tasks/netbox.yml

```yaml
---
- name: Wait for NetBox to be ready
  uri:
    url: "http://localhost:8000"
    status_code: 200
    timeout: 30
  register: netbox_ready
  until: netbox_ready.status == 200
  retries: 30
  delay: 10
  ignore_errors: yes

- name: Create API token for NetBox
  uri:
    url: "http://localhost:8000/api/users/tokens/"
    method: POST
    body_format: json
    body:
      username: "{{ netbox_admin_user }}"
      password: "{{ netbox_admin_password }}"
      description: "Ansible Integration Token"
      expires: null
    status_code: 201
    headers:
      Content-Type: "application/json"
  register: netbox_token
  when: netbox_ready.status == 200

- name: Set NetBox API token fact
  set_fact:
    netbox_api_token: "{{ netbox_token.json.key }}"
  when: netbox_token is defined and netbox_token.json.key is defined
```

## 11. tasks/integration.yml

```yaml
---
- name: Install NetBox inventory script
  copy:
    src: files/scripts/netbox_inventory.py
    dest: /usr/local/bin/netbox_inventory.py
    mode: '0755'

- name: Configure NetBox inventory
  template:
    src: netbox_inventory.cfg.j2
    dest: "{{ netbox_inventory_config }}"
    mode: '0644'

- name: Install Terraform
  unarchive:
    src: "https://releases.hashicorp.com/terraform/{{ terraform_version }}/terraform_{{ terraform_version }}_linux_amd64.zip"
    dest: /usr/local/bin
    remote_src: yes
    mode: '0755'

- name: Create Terraform plugins directory
  file:
    path: "{{ terraform_plugins_dir }}"
    state: directory
    mode: '0755'

- name: Configure AWX integration with NetBox
  uri:
    url: "http://localhost:8052/api/v2/inventory_scripts/"
    method: POST
    body_format: json
    body:
      name: "NetBox Dynamic Inventory"
      script: "{{ lookup('file', '/usr/local/bin/netbox_inventory.py') }}"
      organization: 1
    status_code: 201
    headers:
      Content-Type: "application/json"
  register: awx_inventory_script
  when: awx_ready.status == 200

- name: Create inventory in AWX
  uri:
    url: "http://localhost:8052/api/v2/inventories/"
    method: POST
    body_format: json
    body:
      name: "NetBox Inventory"
      organization: 1
      variables: |
        {
          "netbox_url": "http://netbox:8000",
          "netbox_token": "{{ netbox_api_token }}",
          "group_by": "{{ netbox_inventory_group_by }}",
          "filter": {{ netbox_inventory_filter }}
        }
    status_code: 201
    headers:
      Content-Type: "application/json"
  register: awx_inventory
  when: awx_ready.status == 200 and netbox_api_token is defined

- name: Add inventory source to AWX inventory
  uri:
    url: "http://localhost:8052/api/v2/inventories/{{ awx_inventory.json.id }}/inventory_sources/"
    method: POST
    body_format: json
    body:
      name: "NetBox Source"
      source: "custom"
      source_script: "{{ awx_inventory_script.json.id }}"
      source_vars: |
        {
          "netbox_url": "http://netbox:8000",
          "netbox_token": "{{ netbox_api_token }}",
          "group_by": "{{ netbox_inventory_group_by }}",
          "filter": {{ netbox_inventory_filter }}
        }
    status_code: 201
    headers:
      Content-Type: "application/json"
  when: awx_inventory is defined and awx_inventory_script is defined

- name: Create Terraform project in AWX
  uri:
    url: "http://localhost:8052/api/v2/projects/"
    method: POST
    body_format: json
    body:
      name: "Terraform Projects"
      scm_type: "git"
      scm_url: "https://github.com/example/terraform-repo.git"
      organization: 1
    status_code: 201
    headers:
      Content-Type: "application/json"
  register: awx_terraform_project
  when: awx_ready.status == 200

- name: Create Terraform job template in AWX
  uri:
    url: "http://localhost:8052/api/v2/job_templates/"
    method: POST
    body_format: json
    body:
      name: "Terraform Apply"
      job_type: "run"
      inventory: "{{ awx_inventory.json.id }}"
      project: "{{ awx_terraform_project.json.id }}"
      playbook: "terraform_apply.yml"
      extra_vars: |
        {
          "tf_dir": "{{ terraform_project_dir }}"
        }
    status_code: 201
    headers:
      Content-Type: "application/json"
  when: awx_inventory is defined and awx_terraform_project is defined
```

## 12. tasks/verification.yml

```yaml
---
- name: Verify AWX is accessible
  uri:
    url: "http://{{ awx_hostname }}"
    status_code: 200
    timeout: 30
  register: awx_web_ready
  until: awx_web_ready.status == 200
  retries: 10
  delay: 10

- name: Verify NetBox is accessible
  uri:
    url: "http://{{ netbox_hostname }}"
    status_code: 200
    timeout: 30
  register: netbox_web_ready
  until: netbox_web_ready.status == 200
  retries: 10
  delay: 10

- name: Verify dynamic inventory works
  command: /usr/local/bin/netbox_inventory.py --list
  register: inventory_test
  changed_when: false

- name: Display inventory test result
  debug:
    var: inventory_test.stdout
```

## 13. templates/docker-compose.j2

```yaml
version: '3.7'

services:
  nginx:
    image: nginx:latest
    ports:
      - "{{ nginx_http_port }}:80"
      - "{{ nginx_https_port }}:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/certs:/etc/nginx/certs
    depends_on:
      - awx
      - netbox
    networks:
      - netbox_awx_network

  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: "{{ awx_pg_db }}"
      POSTGRES_USER: "{{ awx_pg_user }}"
      POSTGRES_PASSWORD: "{{ awx_pg_password }}"
    volumes:
      - awx_postgres_data:/var/lib/postgresql/data
    networks:
      - netbox_awx_network

  redis:
    image: redis:latest
    networks:
      - netbox_awx_network

  awx:
    image: ansible/awx:{{ awx_version }}
    hostname: awx
    environment:
      AWX_ADMIN_USER: "{{ awx_admin_user }}"
      AWX_ADMIN_PASSWORD: "{{ awx_admin_password }}"
      DATABASE_HOST: postgres
      DATABASE_PORT: "{{ awx_pg_port }}"
      DATABASE_NAME: "{{ awx_pg_db }}"
      DATABASE_USER: "{{ awx_pg_user }}"
      DATABASE_PASSWORD: "{{ awx_pg_password }}"
      AWX_SECRET_KEY: "{{ awx_secret_key }}"
      AWX_SKIP_MIGRATIONS: "false"
    volumes:
      - "{{ awx_project_dir }}:/var/lib/awx/projects"
    depends_on:
      - postgres
      - redis
    networks:
      - netbox_awx_network

  netbox:
    image: netboxcommunity/netbox:{{ netbox_version }}
    depends_on:
      - netbox-postgres
      - netbox-redis
    environment:
      - DB_NAME="{{ netbox_pg_db }}"
      - DB_USER="{{ netbox_pg_user }}"
      - DB_PASSWORD="{{ netbox_pg_password }}"
      - DB_HOST=netbox-postgres
      - DB_PORT="{{ netbox_pg_port }}"
      - REDIS_HOST=netbox-redis
      - REDIS_PORT="{{ netbox_redis_port }}"
      - SUPERUSER_NAME="{{ netbox_admin_user }}"
      - SUPERUSER_EMAIL="{{ netbox_admin_email }}"
      - SUPERUSER_PASSWORD="{{ netbox_admin_password }}"
      - SECRET_KEY="{{ netbox_secret_key }}"
      - ALLOWED_HOSTS=*
    volumes:
      - netbox-media-files:/opt/netbox/netbox/media
    networks:
      - netbox_awx_network

  netbox-postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB="{{ netbox_pg_db }}"
      - POSTGRES_USER="{{ netbox_pg_user }}"
      - POSTGRES_PASSWORD="{{ netbox_pg_password }}"
    volumes:
      - netbox-postgres-data:/var/lib/postgresql/data
    networks:
      - netbox_awx_network

  netbox-redis:
    image: redis:6
    networks:
      - netbox_awx_network

volumes:
  awx_postgres_data:
  awx_projects:
  netbox-postgres-data:
  netbox-media-files:

networks:
  netbox_awx_network:
    driver: bridge
```

## 14. templates/nginx.conf.j2

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;
}

http {
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    server_tokens off;

    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';

    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    gzip on;
    gzip_disable "msie6";

    upstream awx {
        server awx:8052;
    }

    upstream netbox {
        server netbox:8000;
    }

    server {
        listen 80;
        server_name {{ awx_hostname }} {{ netbox_hostname }};
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name {{ awx_hostname }};

        ssl_certificate {{ ssl_cert_path }};
        ssl_certificate_key {{ ssl_key_path }};

        location / {
            proxy_pass http://awx;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 443 ssl;
        server_name {{ netbox_hostname }};

        ssl_certificate {{ ssl_cert_path }};
        ssl_certificate_key {{ ssl_key_path }};

        location / {
            proxy_pass http://netbox;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

## 15. meta/main.yml

```yaml
---
galaxy_info:
  author: "Your Name"
  description: "Deploy AWX, NetBox, Nginx with Terraform integration"
  company: "Your Company"
  license: "MIT"
  min_ansible_version: "2.9"
  platforms:
    - name: Ubuntu
      versions:
        - 20.04
        - 22.04
    - name: CentOS
      versions:
        - 7
        - 8
  galaxy_tags:
    - awx
    - netbox
    - docker
    - terraform
    - nginx

dependencies: []
```

## Использование роли

1. Создайте playbook `deploy_stack.yml`:

```yaml
---
- hosts: localhost
  become: yes
  vars:
    awx_hostname: "awx.yourdomain.com"
    netbox_hostname: "netbox.yourdomain.com"
    awx_admin_password: "YourSecurePassword123!"
    netbox_admin_password: "YourSecureNetboxPassword123!"
  roles:
    - awx_netbox_stack
```

2. Запустите playbook:

```bash
ansible-playbook deploy_stack.yml
```

3. После развертывания:
   - AWX будет доступен по https://awx.yourdomain.com
   - NetBox будет доступен по https://netbox.yourdomain.com
   - Динамический инвентарь будет настроен для работы с NetBox
   - Terraform будет интегрирован с AWX

Для дальнейшей настройки:
1. Настройте LDAP/AD интеграцию в NetBox (если нужно)
2. Добавьте дополнительные inventory scripts в AWX
3. Настройте credentials в AWX для доступа к инфраструктуре
4. Добавьте свои Terraform проекты в AWX