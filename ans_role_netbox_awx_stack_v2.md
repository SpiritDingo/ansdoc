# Ansible роль для развертывания и настройки Docker-окружения с интеграцией NetBox, AWX и Terraform

## Описание роли

Эта роль развертывает комплексное решение для управления инфраструктурой с использованием:
- Nginx в качестве обратного прокси
- AWX (веб-интерфейс Ansible) для оркестрации
- NetBox как источник истины для инфраструктуры
- Ansible и Terraform для управления конфигурацией
- Динамическую интеграцию инвентаря между компонентами
- Интеграцию с GitLab для управления кодом

## Структура роли

```
roles/docker_infra/
├── defaults
│   └── main.yml
├── files
│   ├── nginx
│   │   └── nginx.conf
│   ├── awx
│   │   └── awx_config.yml
│   ├── netbox
│   │   └── configuration.py
│   └── terraform
│       └── main.tf
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   ├── main.yml
│   ├── docker_compose.yml
│   ├── nginx.yml
│   ├── awx.yml
│   ├── netbox.yml
│   ├── integrations.yml
│   └── gitlab_integration.yml
├── templates
│   ├── docker-compose.yml.j2
│   └── env_files
│       ├── awx.env.j2
│       ├── netbox.env.j2
│       └── postgres.env.j2
└── vars
    └── main.yml
```

## Основные задачи роли

### 1. Установка зависимостей

```yaml
# tasks/main.yml
- name: Install required packages
  apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
  loop:
    - docker-ce
    - docker-ce-cli
    - containerd.io
    - docker-compose-plugin
    - python3-pip
    - git

- name: Install docker-compose via pip
  pip:
    name: docker-compose
    state: present
```

### 2. Настройка Docker Compose

```yaml
# tasks/docker_compose.yml
- name: Create docker-compose directory structure
  file:
    path: "{{ item }}"
    state: directory
  loop:
    - "{{ docker_root }}/nginx"
    - "{{ docker_root }}/awx"
    - "{{ docker_root }}/netbox"
    - "{{ docker_root }}/terraform"

- name: Deploy docker-compose template
  template:
    src: docker-compose.yml.j2
    dest: "{{ docker_root }}/docker-compose.yml"
    owner: root
    group: root
    mode: '0644'

- name: Deploy environment files
  template:
    src: "{{ item.src }}"
    dest: "{{ docker_root }}/{{ item.dest }}"
    owner: root
    group: root
    mode: '0640'
  loop:
    - { src: 'env_files/awx.env.j2', dest: 'awx/awx.env' }
    - { src: 'env_files/netbox.env.j2', dest: 'netbox/netbox.env' }
    - { src: 'env_files/postgres.env.j2', dest: 'postgres/postgres.env' }
```

### 3. Настройка Nginx

```yaml
# tasks/nginx.yml
- name: Deploy Nginx configuration
  template:
    src: nginx/nginx.conf
    dest: "{{ docker_root }}/nginx/nginx.conf"
    owner: root
    group: root
    mode: '0644'

- name: Ensure Nginx is running
  docker_compose:
    project_src: "{{ docker_root }}"
    state: present
    services: nginx
```

### 4. Настройка AWX

```yaml
# tasks/awx.yml
- name: Deploy AWX configuration
  template:
    src: awx/awx_config.yml
    dest: "{{ docker_root }}/awx/awx_config.yml"
    owner: root
    group: root
    mode: '0640'

- name: Initialize AWX
  command: >
    docker-compose -f {{ docker_root }}/docker-compose.yml run --rm awx_task ansible-playbook
    -i install.yml
  args:
    chdir: "{{ docker_root }}/awx"
```

### 5. Настройка NetBox

```yaml
# tasks/netbox.yml
- name: Deploy NetBox configuration
  template:
    src: netbox/configuration.py
    dest: "{{ docker_root }}/netbox/netbox/configuration.py"
    owner: root
    group: root
    mode: '0640'

- name: Run NetBox migrations
  command: >
    docker-compose -f {{ docker_root }}/docker-compose.yml run --rm netbox python3
    manage.py migrate
  args:
    chdir: "{{ docker_root }}/netbox"

- name: Create NetBox superuser
  command: >
    docker-compose -f {{ docker_root }}/docker-compose.yml run --rm netbox python3
    manage.py createsuperuser
  args:
    chdir: "{{ docker_root }}/netbox"
  when: netbox_create_superuser
```

### 6. Интеграция компонентов

```yaml
# tasks/integrations.yml
- name: Configure NetBox as dynamic inventory source in AWX
  uri:
    url: "http://localhost:8052/api/v2/inventory_sources/"
    method: POST
    body_format: json
    body:
      name: "NetBox Dynamic Inventory"
      source: "scm"
      source_path: "https://github.com/netbox-community/ansible-collection-netbox.git"
      source_project: "NetBox Collection"
      source_vars: |
        netbox_url: "http://netbox:8000"
        netbox_token: "{{ netbox_api_token }}"
        group_by:
          - device_roles
          - tags
          - sites
        validate_certs: false
    status_code: 201
    headers:
      Content-Type: "application/json"
      Authorization: "Bearer {{ awx_admin_token }}"

- name: Configure Terraform integration in AWX
  uri:
    url: "http://localhost:8052/api/v2/credentials/"
    method: POST
    body_format: json
    body:
      name: "Terraform Cloud"
      credential_type: 20  # Terraform Cloud API Token
      inputs:
        api_token: "{{ terraform_cloud_token }}"
    status_code: 201
    headers:
      Content-Type: "application/json"
      Authorization: "Bearer {{ awx_admin_token }}"
```

### 7. Интеграция с GitLab

```yaml
# tasks/gitlab_integration.yml
- name: Add GitLab projects to AWX
  uri:
    url: "http://localhost:8052/api/v2/projects/"
    method: POST
    body_format: json
    body:
      name: "{{ item.name }}"
      description: "{{ item.description }}"
      scm_type: "git"
      scm_url: "{{ gitlab_url }}/{{ item.path }}.git"
      scm_branch: "{{ item.branch | default('main') }}"
      credential: "{{ gitlab_credential_id }}"
      organization: "Default"
    status_code: 201
    headers:
      Content-Type: "application/json"
      Authorization: "Bearer {{ awx_admin_token }}"
  loop: "{{ gitlab_projects }}"
  register: awx_projects
```

## Шаблон docker-compose.yml.j2

```jinja2
version: '3.8'

services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx/certs:/etc/nginx/certs
    networks:
      - infra_network
    restart: unless-stopped

  postgres:
    image: postgres:13
    container_name: postgres
    env_file:
      - ./postgres/postgres.env
    volumes:
      - ./postgres/data:/var/lib/postgresql/data
    networks:
      - infra_network
    restart: unless-stopped

  redis:
    image: redis:6
    container_name: redis
    networks:
      - infra_network
    restart: unless-stopped

  netbox:
    image: netboxcommunity/netbox:{{ netbox_version }}
    container_name: netbox
    depends_on:
      - postgres
      - redis
    env_file:
      - ./netbox/netbox.env
    volumes:
      - ./netbox/configuration.py:/etc/netbox/config/configuration.py
    networks:
      - infra_network
    restart: unless-stopped

  awx_web:
    image: ansible/awx:{{ awx_version }}
    container_name: awx_web
    depends_on:
      - postgres
      - redis
    env_file:
      - ./awx/awx.env
    ports:
      - "8052:8052"
    networks:
      - infra_network
    restart: unless-stopped

  awx_task:
    image: ansible/awx:{{ awx_version }}
    container_name: awx_task
    depends_on:
      - postgres
      - redis
      - awx_web
    env_file:
      - ./awx/awx.env
    volumes:
      - ./awx/awx_config.yml:/etc/awx/conf.d/awx_config.yml
    networks:
      - infra_network
    restart: unless-stopped

  terraform:
    image: hashicorp/terraform:{{ terraform_version }}
    container_name: terraform
    volumes:
      - ./terraform:/terraform
    networks:
      - infra_network
    restart: unless-stopped

networks:
  infra_network:
    driver: bridge
```

## Переменные по умолчанию

```yaml
# defaults/main.yml
docker_root: "/opt/docker_infra"
netbox_version: "v3.3.0"
awx_version: "19.4.0"
terraform_version: "1.1.0"

netbox_create_superuser: true
netbox_superuser_email: "admin@example.com"
netbox_superuser_password: "ChangeMe123"
netbox_allowed_hosts: "['*']"
netbox_secret_key: "generate_me"

awx_admin_user: "admin"
awx_admin_password: "ChangeMe123"
awx_secret_key: "generate_me"

gitlab_url: "https://gitlab.example.com"
gitlab_credential_id: 1  # Должен быть создан заранее

gitlab_projects:
  - name: "Infrastructure Ansible"
    path: "infra/ansible"
    description: "Ansible playbooks for infrastructure"
  - name: "Terraform Modules"
    path: "infra/terraform"
    description: "Terraform modules for cloud infrastructure"
```

## Использование роли

1. Создайте playbook:

```yaml
# deploy_infra.yml
- hosts: infra_servers
  become: yes
  roles:
    - role: docker_infra
      vars:
        netbox_secret_key: "{{ vault_netbox_secret_key }}"
        awx_secret_key: "{{ vault_awx_secret_key }}"
        netbox_superuser_password: "{{ vault_netbox_admin_password }}"
        awx_admin_password: "{{ vault_awx_admin_password }}"
        gitlab_credential_id: "{{ vault_gitlab_token_id }}"
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory deploy_infra.yml --ask-vault-pass
```

## Дальнейшая настройка

После развертывания необходимо:
1. Настроить SSL в Nginx
2. Настроить брандмауэрные правила
3. Настроить резервное копирование баз данных
4. Настроить мониторинг компонентов
5. Настроить аутентификацию (LDAP/OAuth2) для AWX и NetBox

Это комплексное решение обеспечит полный цикл управления инфраструктурой с централизованным источником истины (NetBox), оркестрацией (AWX), управлением конфигурацией (Ansible) и инфраструктурой как код (Terraform).