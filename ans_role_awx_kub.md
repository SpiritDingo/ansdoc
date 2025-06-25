# Ansible роли для установки и настройки AWX в Minikube с интеграциями

Вот структура для набора ролей Ansible, которые устанавливают и настраивают AWX в Minikube с интеграцией GitLab и NetBox.

## Основная структура проекта

```
awx-minikube-deploy/
├── roles/
│   ├── awx_minikube/
│   ├── gitlab_integration/
│   ├── netbox_inventory/
├── inventory/
│   └── hosts
└── playbooks/
    ├── deploy_awx.yml
    ├── configure_gitlab.yml
    └── configure_netbox.yml
```

## 1. Роль `awx_minikube` (установка AWX в Minikube)

`roles/awx_minikube/tasks/main.yml`:

```yaml
---
- name: Установка зависимостей
  become: yes
  apt:
    name:
      - docker.io
      - docker-compose
      - python3-pip
      - virtualenv
    state: present

- name: Установка kubectl
  become: yes
  apt:
    name: kubectl
    state: present

- name: Установка Minikube
  become: yes
  shell: |
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
    install minikube-linux-amd64 /usr/local/bin/minikube

- name: Запуск Minikube
  become: yes
  shell: minikube start --driver=docker

- name: Установка оператора AWX
  k8s:
    state: present
    definition:
      apiVersion: operators.coreos.com/v1alpha1
      kind: Subscription
      metadata:
        name: awx-operator
        namespace: operators
      spec:
        channel: stable
        name: awx-operator
        source: operatorhubio-catalog
        sourceNamespace: olm

- name: Создание namespace для AWX
  k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: awx

- name: Развертывание AWX
  k8s:
    state: present
    definition:
      apiVersion: awx.ansible.com/v1beta1
      kind: AWX
      metadata:
        name: awx
        namespace: awx
      spec:
        service_type: nodeport
        ingress_type: none
```

## 2. Роль `gitlab_integration` (интеграция с GitLab)

`roles/gitlab_integration/tasks/main.yml`:

```yaml
---
- name: Получение токена администратора AWX
  shell: |
    kubectl get secret -n awx awx-admin-password -o jsonpath="{.data.password}" | base64 --decode
  register: awx_admin_token
  delegate_to: localhost

- name: Установка AWX CLI
  pip:
    name: awxkit
    state: present

- name: Настройка подключения к GitLab в AWX
  uri:
    url: "http://{{ awx_host }}:{{ awx_port }}/api/v2/projects/"
    method: POST
    headers:
      Content-Type: "application/json"
      Authorization: "Bearer {{ awx_admin_token.stdout }}"
    body:
      name: "GitLab Integration"
      scm_type: git
      scm_url: "{{ gitlab_repo_url }}"
      scm_branch: "{{ gitlab_branch | default('main') }}"
      credential: "{{ gitlab_credential_id }}"
    body_format: json
    status_code: 201
  register: gitlab_project
  when: gitlab_repo_url is defined
```

## 3. Роль `netbox_inventory` (динамический инвентарь NetBox)

`roles/netbox_inventory/tasks/main.yml`:

```yaml
---
- name: Создание директории для плагинов
  file:
    path: /etc/ansible/plugins/inventory
    state: directory

- name: Установка плагина NetBox
  get_url:
    url: https://raw.githubusercontent.com/netbox-community/ansible_modules/develop/plugins/inventory/netbox.py
    dest: /etc/ansible/plugins/inventory/netbox.py
    mode: '0755'

- name: Создание конфигурационного файла для инвентаря NetBox
  template:
    src: netbox_inventory.j2
    dest: /etc/ansible/netbox_inventory.yml
    mode: '0644'

- name: Создание credential в AWX для NetBox
  uri:
    url: "http://{{ awx_host }}:{{ awx_port }}/api/v2/credentials/"
    method: POST
    headers:
      Content-Type: "application/json"
      Authorization: "Bearer {{ awx_admin_token.stdout }}"
    body:
      name: "NetBox API Token"
      credential_type: 18  # API Token
      inputs:
        api_token: "{{ netbox_api_token }}"
    body_format: json
    status_code: 201
  register: netbox_credential
  when: netbox_api_token is defined

- name: Создание источника инвентаря NetBox в AWX
  uri:
    url: "http://{{ awx_host }}:{{ awx_port }}/api/v2/inventory_sources/"
    method: POST
    headers:
      Content-Type: "application/json"
      Authorization: "Bearer {{ awx_admin_token.stdout }}"
    body:
      name: "NetBox Dynamic Inventory"
      source: "scm"
      source_path: "/etc/ansible/netbox_inventory.yml"
      source_project: "Dynamic Inventories"
      credential: "{{ netbox_credential.json.id }}"
    body_format: json
    status_code: 201
  when: netbox_credential is defined and netbox_credential.json.id is defined
```

## Playbooks

`playbooks/deploy_awx.yml`:

```yaml
---
- hosts: awx_servers
  roles:
    - awx_minikube
```

`playbooks/configure_gitlab.yml`:

```yaml
---
- hosts: awx_servers
  vars:
    awx_host: "localhost"
    awx_port: "30080"
  roles:
    - gitlab_integration
```

`playbooks/configure_netbox.yml`:

```yaml
---
- hosts: awx_servers
  vars:
    awx_host: "localhost"
    awx_port: "30080"
    netbox_url: "http://netbox.example.com"
    netbox_api_token: "your_api_token_here"
  roles:
    - netbox_inventory
```

## Шаблон конфигурации NetBox

`roles/netbox_inventory/templates/netbox_inventory.j2`:

```yaml
---
plugin: netbox
api_endpoint: {{ netbox_url }}
validate_certs: False
config_context: True
group_by:
  - device_roles
query_filters:
  - status: active
```

## Инвентарь

`inventory/hosts`:

```ini
[awx_servers]
localhost ansible_connection=local

[gitlab]
gitlab.example.com

[netbox]
netbox.example.com
```

## Переменные

Вы можете определить переменные в `group_vars/` или `host_vars/`, например:

`group_vars/awx_servers.yml`:

```yaml
---
awx_host: "localhost"
awx_port: "30080"
gitlab_repo_url: "https://gitlab.example.com/your/repo.git"
gitlab_credential_id: 1
netbox_url: "http://netbox.example.com"
netbox_api_token: "your_api_token_here"
```

## Запуск

```bash
# Установка AWX в Minikube
ansible-playbook -i inventory/hosts playbooks/deploy_awx.yml

# Настройка интеграции с GitLab
ansible-playbook -i inventory/hosts playbooks/configure_gitlab.yml

# Настройка динамического инвентаря NetBox
ansible-playbook -i inventory/hosts playbooks/configure_netbox.yml
```

Этот набор ролей обеспечит полную установку AWX в Minikube с интеграцией GitLab для управления проектами и NetBox для динамического инвентаря.