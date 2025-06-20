Вот структура Ansible ролей для ваших задач:

### 1. Роль для установки и настройки Nginx и AWX в Docker-compose (`nginx_awx_docker`)

**Структура роли:**
```
roles/nginx_awx_docker/
├── defaults
│   └── main.yml       # переменные по умолчанию
├── files
│   ├── awx_docker-compose.yml  # шаблон docker-compose для AWX
│   └── nginx.conf     # конфиг nginx
├── handlers
│   └── main.yml       # обработчики для перезагрузки сервисов
├── meta
│   └── main.yml       # метаданные роли
├── tasks
│   └── main.yml       # основные задачи
└── templates
    └── nginx-awx.conf.j2  # шаблон конфига nginx для AWX
```

**tasks/main.yml:**
```yaml
---
- name: Установка зависимостей
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - docker-ce
    - docker-compose-plugin
    - python3-docker

- name: Создание директорий для AWX
  file:
    path: "{{ item }}"
    state: directory
  loop:
    - /opt/awx
    - /opt/awx/postgres_data

- name: Копирование docker-compose для AWX
  copy:
    src: files/awx_docker-compose.yml
    dest: /opt/awx/docker-compose.yml

- name: Запуск AWX в docker-compose
  community.docker.docker_compose:
    project_src: /opt/awx
    state: present

- name: Установка и настройка Nginx
  template:
    src: templates/nginx-awx.conf.j2
    dest: /etc/nginx/conf.d/awx.conf
  notify: restart nginx

- name: Проверка работы AWX
  uri:
    url: "http://localhost:8052"
    return_content: yes
  register: awx_status
  until: awx_status.status == 200
  retries: 10
  delay: 30
```

### 2. Роль для интеграции GitLab и AWX (`gitlab_awx_integration`)

**Структура роли:**
```
roles/gitlab_awx_integration/
├── defaults
│   └── main.yml       # переменные интеграции
├── tasks
│   └── main.yml       # задачи интеграции
└── templates
    └── awx_credential.yml.j2  # шаблон для учетных данных
```

**tasks/main.yml:**
```yaml
---
- name: Создание учетных данных GitLab в AWX
  uri:
    url: "{{ awx_url }}/api/v2/credentials/"
    method: POST
    headers:
      Content-Type: "application/json"
      Authorization: "Bearer {{ awx_token }}"
    body_format: json
    body: "{{ lookup('template', 'templates/awx_credential.yml.j2') | from_json }}"
    status_code: 201
```

### 3. Роль для динамического инвентаря NetBox и AWX (`netbox_awx_inventory`)

**Структура роли:**
```
roles/netbox_awx_inventory/
├── defaults
│   └── main.yml       # переменные подключения
├── files
│   └── netbox_inventory.py  # скрипт инвентаря
├── tasks
│   └── main.yml       # задачи настройки
└── templates
    └── awx_inventory.yml.j2  # шаблон инвентаря AWX
```

**tasks/main.yml:**
```yaml
---
- name: Установка зависимостей NetBox
  pip:
    name:
      - pynetbox
      - requests

- name: Копирование скрипта динамического инвентаря
  copy:
    src: files/netbox_inventory.py
    dest: /usr/local/bin/netbox_inventory.py
    mode: 0755

- name: Создание инвентаря в AWX
  uri:
    url: "{{ awx_url }}/api/v2/inventories/"
    method: POST
    headers:
      Content-Type: "application/json"
      Authorization: "Bearer {{ awx_token }}"
    body_format: json
    body: "{{ lookup('template', 'templates/awx_inventory.yml.j2') | from_json }}"
    status_code: 201
```

### 4. Роль для установки NetBox и Nginx в Docker-compose (`netbox_nginx_docker`)

**Структура роли:**
```
roles/netbox_nginx_docker/
├── defaults
│   └── main.yml       # переменные NetBox
├── files
│   ├── netbox-docker-compose.yml
│   └── netbox-nginx.conf
├── tasks
│   └── main.yml       # задачи установки
└── templates
    └── nginx-netbox.conf.j2
```

**tasks/main.yml:**
```yaml
---
- name: Создание директорий для NetBox
  file:
    path: "{{ item }}"
    state: directory
  loop:
    - /opt/netbox
    - /opt/netbox/postgres_data
    - /opt/netbox/redis_data

- name: Копирование docker-compose для NetBox
  copy:
    src: files/netbox-docker-compose.yml
    dest: /opt/netbox/docker-compose.yml

- name: Запуск NetBox в docker-compose
  community.docker.docker_compose:
    project_src: /opt/netbox
    state: present

- name: Настройка Nginx для NetBox
  template:
    src: templates/nginx-netbox.conf.j2
    dest: /etc/nginx/conf.d/netbox.conf
  notify: restart nginx
```

### Пример playbook для развертывания всей инфраструктуры:

```yaml
---
- name: Развертывание AWX и Nginx
  hosts: awx_host
  roles:
    - nginx_awx_docker

- name: Развертывание NetBox и Nginx
  hosts: netbox_host
  roles:
    - netbox_nginx_docker

- name: Настройка интеграции GitLab и AWX
  hosts: awx_host
  roles:
    - gitlab_awx_integration

- name: Настройка динамического инвентаря
  hosts: awx_host
  roles:
    - netbox_awx_inventory
```

Для каждой роли нужно будет настроить соответствующие переменные в `defaults/main.yml` или передавать их через playbook.