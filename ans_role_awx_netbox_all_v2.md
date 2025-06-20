Вот структура Ansible ролей для ваших задач. Я предоставлю основные файлы и структуру каждой роли.

### 1. Роль для установки Nginx и AWX в Docker Compose (`nginx_awx_docker`)

Структура роли:
```
roles/nginx_awx_docker/
├── defaults
│   └── main.yml
├── files
│   ├── awx_docker-compose.yml.j2
│   └── nginx_awx.conf.j2
├── tasks
│   └── main.yml
└── templates
    └── nginx_awx.conf.j2
```

`tasks/main.yml`:
```yaml
---
- name: Create directories for AWX
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
  loop:
    - "{{ awx_project_dir }}"
    - "{{ awx_postgres_data_dir }}"
    - "{{ awx_nginx_conf_dir }}"

- name: Deploy AWX docker-compose file
  ansible.builtin.template:
    src: awx_docker-compose.yml.j2
    dest: "{{ awx_project_dir }}/docker-compose.yml"
    mode: '0644'

- name: Deploy Nginx config for AWX
  ansible.builtin.template:
    src: nginx_awx.conf.j2
    dest: "{{ awx_nginx_conf_dir }}/awx.conf"
    mode: '0644'
  notify: Restart Nginx

- name: Start AWX containers
  community.docker.docker_compose:
    project_src: "{{ awx_project_dir }}"
    build: yes
    pull: yes
    state: present

- name: Ensure Nginx is running
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: yes
```

### 2. Роль для интеграции GitLab и AWX (`gitlab_awx_integration`)

Структура роли:
```
roles/gitlab_awx_integration/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
└── templates
    └── gitlab_webhook.sh.j2
```

`tasks/main.yml`:
```yaml
---
- name: Install required Python libraries
  pip:
    name:
      - awxkit
      - requests
    state: present

- name: Create GitLab credentials in AWX
  uri:
    url: "{{ awx_url }}/api/v2/credentials/"
    method: POST
    headers:
      Authorization: "Bearer {{ awx_admin_token }}"
      Content-Type: "application/json"
    body_format: json
    body:
      name: "GitLab Credential"
      credential_type: 2  # Source Control
      organization: 1
      inputs:
        username: "{{ gitlab_username }}"
        password: "{{ gitlab_token }}"
    status_code: 201

- name: Create projects in AWX from GitLab
  uri:
    url: "{{ awx_url }}/api/v2/projects/"
    method: POST
    headers:
      Authorization: "Bearer {{ awx_admin_token }}"
      Content-Type: "application/json"
    body_format: json
    body:
      name: "{{ item.name }}"
      scm_type: git
      scm_url: "{{ item.url }}"
      scm_branch: "{{ item.branch | default('main') }}"
      credential: "{{ gitlab_credential_id }}"
  loop: "{{ gitlab_projects }}"
  register: awx_projects

- name: Set up GitLab webhooks
  uri:
    url: "{{ gitlab_url }}/api/v4/projects/{{ item.project_id }}/hooks"
    method: POST
    headers:
      PRIVATE-TOKEN: "{{ gitlab_admin_token }}"
      Content-Type: "application/json"
    body_format: json
    body:
      url: "{{ awx_webhook_url }}"
      push_events: true
      token: "{{ awx_webhook_token }}"
      enable_ssl_verification: false
  loop: "{{ gitlab_projects }}"
  when: item.project_id is defined
```

### 3. Роль для динамического инвентаря NetBox и AWX (`netbox_awx_inventory`)

Структура роли:
```
roles/netbox_awx_inventory/
├── defaults
│   └── main.yml
├── files
│   └── netbox_inventory.py
├── tasks
│   └── main.yml
└── templates
    └── netbox_awx_inventory.yml.j2
```

`tasks/main.yml`:
```yaml
---
- name: Install NetBox inventory script dependencies
  pip:
    name:
      - pynetbox
      - requests

- name: Deploy NetBox inventory script
  ansible.builtin.copy:
    src: netbox_inventory.py
    dest: "{{ netbox_inventory_script_path }}"
    mode: '0755'

- name: Create inventory in AWX
  uri:
    url: "{{ awx_url }}/api/v2/inventories/"
    method: POST
    headers:
      Authorization: "Bearer {{ awx_admin_token }}"
      Content-Type: "application/json"
    body_format: json
    body:
      name: "NetBox Dynamic Inventory"
      organization: 1
    register: inventory_result

- name: Create inventory source for NetBox
  uri:
    url: "{{ awx_url }}/api/v2/inventory_sources/"
    method: POST
    headers:
      Authorization: "Bearer {{ awx_admin_token }}"
      Content-Type: "application/json"
    body_format: json
    body:
      name: "NetBox Source"
      inventory: "{{ inventory_result.json.id }}"
      source: "scm"
      source_path: "{{ netbox_inventory_script_path }}"
      source_project: "{{ netbox_project_id }}"
      source_vars: "{{ netbox_inventory_vars | to_json }}"
      update_on_launch: true
```

### 4. Роль для установки NetBox и Nginx в Docker Compose (`netbox_nginx_docker`)

Структура роли:
```
roles/netbox_nginx_docker/
├── defaults
│   └── main.yml
├── files
│   └── netbox_docker-compose.yml.j2
├── tasks
│   └── main.yml
└── templates
    └── nginx_netbox.conf.j2
```

`tasks/main.yml`:
```yaml
---
- name: Create directories for NetBox
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
  loop:
    - "{{ netbox_project_dir }}"
    - "{{ netbox_postgres_data_dir }}"
    - "{{ netbox_nginx_conf_dir }}"

- name: Deploy NetBox docker-compose file
  ansible.builtin.template:
    src: netbox_docker-compose.yml.j2
    dest: "{{ netbox_project_dir }}/docker-compose.yml"
    mode: '0644'

- name: Deploy Nginx config for NetBox
  ansible.builtin.template:
    src: nginx_netbox.conf.j2
    dest: "{{ netbox_nginx_conf_dir }}/netbox.conf"
    mode: '0644'
  notify: Restart Nginx

- name: Start NetBox containers
  community.docker.docker_compose:
    project_src: "{{ netbox_project_dir }}"
    build: yes
    pull: yes
    state: present

- name: Run NetBox migrations
  community.docker.docker_container_exec:
    container: netbox
    command: /opt/netbox/netbox/manage.py migrate --no-input

- name: Create NetBox superuser
  community.docker.docker_container_exec:
    container: netbox
    command: /opt/netbox/netbox/manage.py createsuperuser --no-input
    env:
      DJANGO_SUPERUSER_USERNAME: "{{ netbox_admin_user }}"
      DJANGO_SUPERUSER_PASSWORD: "{{ netbox_admin_pass }}"
      DJANGO_SUPERUSER_EMAIL: "{{ netbox_admin_email }}"
```

### Playbook для использования этих ролей

`main.yml`:
```yaml
---
- name: Deploy Nginx and AWX in Docker
  hosts: awx_host
  roles:
    - nginx_awx_docker

- name: Deploy NetBox and Nginx in Docker
  hosts: netbox_host
  roles:
    - netbox_nginx_docker

- name: Configure GitLab and AWX integration
  hosts: awx_host
  roles:
    - gitlab_awx_integration

- name: Configure NetBox dynamic inventory for AWX
  hosts: awx_host
  roles:
    - netbox_awx_inventory
```

Для каждой роли вам нужно будет определить соответствующие переменные в `defaults/main.yml` или передать их через inventory/extra-vars.

Все роли используют модуль `community.docker.docker_compose`, который должен быть установлен:
```bash
ansible-galaxy collection install community.docker
```

Также убедитесь, что на целевых хостах установлены Docker и Docker Compose, как вы указали в требованиях.



Давайте детально разберём каждую Ansible-роль, включая структуру, основные задачи, переменные и примеры конфигурационных файлов.

---

## **1. Роль `nginx_awx_docker`: Установка Nginx + AWX в Docker Compose**

### **Структура роли**
```
roles/nginx_awx_docker/
├── defaults/           # Переменные по умолчанию
│   └── main.yml
├── tasks/              # Основные задачи
│   └── main.yml
├── templates/          # Jinja2-шаблоны
│   ├── awx_docker-compose.yml.j2
│   └── nginx_awx.conf.j2
└── vars/               # Дополнительные переменные (опционально)
    └── main.yml
```

### **Основные задачи (`tasks/main.yml`)**
```yaml
---
- name: "Создание директорий для AWX"
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ awx_data_dir }}/postgres"
    - "{{ awx_data_dir }}/projects"
    - "{{ awx_data_dir }}/nginx"

- name: "Развёртывание docker-compose для AWX"
  template:
    src: "awx_docker-compose.yml.j2"
    dest: "{{ awx_data_dir }}/docker-compose.yml"
  notify: "Запуск контейнеров AWX"

- name: "Развёртывание конфига Nginx для AWX"
  template:
    src: "nginx_awx.conf.j2"
    dest: "/etc/nginx/conf.d/awx.conf"
  notify: "Перезапуск Nginx"

- name: "Запуск контейнеров AWX"
  community.docker.docker_compose:
    project_src: "{{ awx_data_dir }}"
    build: yes
    pull: yes
    state: present
```

### **Шаблон `awx_docker-compose.yml.j2`**
```yaml
version: '3.7'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: "{{ awx_db_user }}"
      POSTGRES_PASSWORD: "{{ awx_db_password }}"
    volumes:
      - "{{ awx_data_dir }}/postgres:/var/lib/postgresql/data"

  awx:
    image: ansible/awx:latest
    depends_on:
      - postgres
    environment:
      AWX_POSTGRES_USER: "{{ awx_db_user }}"
      AWX_POSTGRES_PASSWORD: "{{ awx_db_password }}"
    ports:
      - "8052:8052"
```

### **Шаблон `nginx_awx.conf.j2`**
```nginx
server {
    listen 80;
    server_name awx.example.com;

    location / {
        proxy_pass http://localhost:8052;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### **Переменные (`defaults/main.yml`)**
```yaml
awx_data_dir: "/opt/awx"
awx_db_user: "awx"
awx_db_password: "secure_password"
```

---

## **2. Роль `gitlab_awx_integration`: Интеграция GitLab и AWX**

### **Структура роли**
```
roles/gitlab_awx_integration/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
└── templates/
    └── gitlab_webhook.sh.j2
```

### **Основные задачи (`tasks/main.yml`)**
```yaml
---
- name: "Создание проекта в AWX из GitLab"
  uri:
    url: "{{ awx_api_url }}/api/v2/projects/"
    method: POST
    headers:
      Authorization: "Bearer {{ awx_api_token }}"
      Content-Type: "application/json"
    body_format: json
    body:
      name: "{{ item.name }}"
      scm_type: git
      scm_url: "{{ item.repo_url }}"
      scm_branch: "{{ item.branch | default('main') }}"
  loop: "{{ gitlab_projects }}"

- name: "Создание Webhook в GitLab"
  uri:
    url: "{{ gitlab_api_url }}/projects/{{ item.id }}/hooks"
    method: POST
    headers:
      PRIVATE-TOKEN: "{{ gitlab_api_token }}"
    body_format: json
    body:
      url: "{{ awx_webhook_url }}"
      push_events: true
  loop: "{{ gitlab_projects }}"
```

### **Переменные (`defaults/main.yml`)**
```yaml
awx_api_url: "http://awx.example.com"
awx_api_token: "your_awx_token"
gitlab_api_url: "https://gitlab.example.com/api/v4"
gitlab_api_token: "your_gitlab_token"
gitlab_projects:
  - name: "ansible-playbooks"
    repo_url: "git@gitlab.example.com:devops/ansible-playbooks.git"
    branch: "main"
    id: "123"  # GitLab Project ID
```

---

## **3. Роль `netbox_awx_inventory`: Динамический инвентарь NetBox → AWX**

### **Структура роли**
```
roles/netbox_awx_inventory/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
└── files/
    └── netbox_inventory.py
```

### **Основные задачи (`tasks/main.yml`)**
```yaml
---
- name: "Копирование скрипта динамического инвентаря"
  copy:
    src: "netbox_inventory.py"
    dest: "/usr/local/bin/netbox_inventory.py"
    mode: "0755"

- name: "Создание Inventory в AWX"
  uri:
    url: "{{ awx_api_url }}/api/v2/inventories/"
    method: POST
    headers:
      Authorization: "Bearer {{ awx_api_token }}"
    body_format: json
    body:
      name: "NetBox Dynamic Inventory"
      organization: 1

- name: "Добавление источника инвентаря"
  uri:
    url: "{{ awx_api_url }}/api/v2/inventory_sources/"
    method: POST
    headers:
      Authorization: "Bearer {{ awx_api_token }}"
    body_format: json
    body:
      name: "NetBox Source"
      source: "script"
      script: "netbox_inventory.py"
      inventory: "{{ inventory_id }}"
```

### **Пример `netbox_inventory.py`**
```python
#!/usr/bin/env python3
import requests
import json

NETBOX_URL = "http://netbox.example.com"
NETBOX_TOKEN = "your_netbox_token"

response = requests.get(
    f"{NETBOX_URL}/api/dcim/devices/",
    headers={"Authorization": f"Token {NETBOX_TOKEN}"}
)
devices = response.json()["results"]

inventory = {
    "_meta": {"hostvars": {}},
    "all": {"hosts": []}
}

for device in devices:
    inventory["all"]["hosts"].append(device["name"])
    inventory["_meta"]["hostvars"][device["name"]] = {
        "ansible_host": device["primary_ip"]["address"].split("/")[0]
    }

print(json.dumps(inventory))
```

---

## **4. Роль `netbox_nginx_docker`: Установка NetBox + Nginx в Docker**

### **Структура роли**
```
roles/netbox_nginx_docker/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
└── templates/
    ├── netbox_docker-compose.yml.j2
    └── nginx_netbox.conf.j2
```

### **Основные задачи (`tasks/main.yml`)**
```yaml
---
- name: "Развёртывание docker-compose для NetBox"
  template:
    src: "netbox_docker-compose.yml.j2"
    dest: "/opt/netbox/docker-compose.yml"
  notify: "Запуск NetBox"

- name: "Настройка Nginx для NetBox"
  template:
    src: "nginx_netbox.conf.j2"
    dest: "/etc/nginx/conf.d/netbox.conf"
  notify: "Перезапуск Nginx"

- name: "Запуск NetBox"
  community.docker.docker_compose:
    project_src: "/opt/netbox"
    build: yes
    pull: yes
    state: present
```

### **Шаблон `netbox_docker-compose.yml.j2`**
```yaml
version: '3.7'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: netbox
      POSTGRES_PASSWORD: "{{ netbox_db_password }}"

  netbox:
    image: netboxcommunity/netbox:latest
    depends_on:
      - postgres
    environment:
      DB_NAME: netbox
      DB_USER: netbox
      DB_PASSWORD: "{{ netbox_db_password }}"
    ports:
      - "8000:8000"
```

### **Шаблон `nginx_netbox.conf.j2`**
```nginx
server {
    listen 80;
    server_name netbox.example.com;

    location / {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
    }
}
```

---

## **Итог**
- Все роли используют **Jinja2-шаблоны** для гибкой настройки.
- **Docker Compose** управляется через модуль `community.docker.docker_compose`.
- **AWX API** используется для автоматизации задач.
- **Динамический инвентарь** NetBox позволяет автоматически обновлять список хостов.

Можно расширить роли, добавив обработку ошибок, тегирование задач и дополнительные настройки безопасности (TLS, аутентификация).