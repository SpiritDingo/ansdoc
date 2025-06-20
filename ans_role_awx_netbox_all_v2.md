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