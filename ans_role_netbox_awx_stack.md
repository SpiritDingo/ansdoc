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