# Роль Ansible для установки и настройки Docker Compose окружения

Эта роль устанавливает и настраивает следующие сервисы в Docker Compose:
- Nginx (как reverse proxy)
- NetBox (IPAM/DCIM)
- AWX (веб-интерфейс Ansible)
- Ansible (локально)
- Настраивает динамическую инвентаризацию AWX из NetBox

## Структура роли

```
netbox-awx-docker/
├── defaults/
│   └── main.yml          # Переменные по умолчанию
├── files/
│   ├── nginx/            # Конфиги Nginx
│   ├── netbox/           # Конфиги NetBox
│   └── awx/              # Конфиги AWX
├── tasks/
│   └── main.yml          # Основные задачи
├── templates/
│   ├── docker-compose.yml.j2  # Шаблон docker-compose
│   └── netbox_awx_inventory.yml.j2  # Шаблон инвентаризации
└── vars/
    └── main.yml          # Основные переменные
```

## Основные задачи (tasks/main.yml)

```yaml
---
- name: Установка зависимостей
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

- name: Установка Ansible
  pip:
    name: ansible
    state: present

- name: Добавление пользователя в группу docker
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Создание директорий для volumes
  file:
    path: "{{ item }}"
    state: directory
    mode: 0755
  loop:
    - "{{ nginx_config_dir }}"
    - "{{ netbox_data_dir }}"
    - "{{ awx_data_dir }}"

- name: Копирование конфигураций Nginx
  copy:
    src: "files/nginx/"
    dest: "{{ nginx_config_dir }}"
    mode: 0644

- name: Копирование конфигураций NetBox
  copy:
    src: "files/netbox/"
    dest: "{{ netbox_data_dir }}/config"
    mode: 0644

- name: Копирование конфигураций AWX
  copy:
    src: "files/awx/"
    dest: "{{ awx_data_dir }}/config"
    mode: 0644

- name: Развертывание docker-compose.yml
  template:
    src: docker-compose.yml.j2
    dest: "{{ project_dir }}/docker-compose.yml"
    mode: 0644

- name: Запуск сервисов в Docker Compose
  community.docker.docker_compose:
    project_src: "{{ project_dir }}"
    build: yes
    pull: yes
    recreate: always
    restart: yes

- name: Настройка динамической инвентаризации AWX из NetBox
  template:
    src: netbox_awx_inventory.yml.j2
    dest: "{{ awx_inventory_script_path }}"
    mode: 0755

- name: Создание плейбука для тестирования интеграции
  copy:
    content: |
      ---
      - name: Test NetBox integration
        hosts: all
        gather_facts: no
        tasks:
          - debug:
              msg: "Host {{ inventory_hostname }} is alive"
    dest: "{{ playbooks_dir }}/test_netbox_integration.yml"
    mode: 0644
```

## Шаблон docker-compose.yml.j2

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "{{ nginx_config_dir }}:/etc/nginx/conf.d"
      - "./certs:/etc/nginx/certs"
    networks:
      - netbox_awx_network
    restart: unless-stopped

  netbox:
    image: netboxcommunity/netbox:latest
    container_name: netbox
    depends_on:
      - netbox-postgres
      - netbox-redis
      - netbox-redis-cache
    environment:
      - ALLOWED_HOSTS=netbox.local
      - DB_NAME=netbox
      - DB_USER=netbox
      - DB_PASSWORD=netbox
      - DB_HOST=netbox-postgres
      - DB_PORT=5432
      - REDIS_HOST=netbox-redis
      - REDIS_PORT=6379
      - REDIS_CACHE_HOST=netbox-redis-cache
      - REDIS_CACHE_PORT=6379
      - SUPERUSER_NAME=admin
      - SUPERUSER_EMAIL=admin@example.com
      - SUPERUSER_PASSWORD=admin
    volumes:
      - "{{ netbox_data_dir }}/config:/etc/netbox/config"
      - "{{ netbox_data_dir }}/reports:/etc/netbox/reports"
      - "{{ netbox_data_dir }}/scripts:/etc/netbox/scripts"
    networks:
      - netbox_awx_network
    restart: unless-stopped

  netbox-postgres:
    image: postgres:14
    container_name: netbox-postgres
    environment:
      - POSTGRES_DB=netbox
      - POSTGRES_USER=netbox
      - POSTGRES_PASSWORD=netbox
    volumes:
      - "{{ netbox_data_dir }}/postgres:/var/lib/postgresql/data"
    networks:
      - netbox_awx_network
    restart: unless-stopped

  netbox-redis:
    image: redis:6
    container_name: netbox-redis
    networks:
      - netbox_awx_network
    restart: unless-stopped

  netbox-redis-cache:
    image: redis:6
    container_name: netbox-redis-cache
    networks:
      - netbox_awx_network
    restart: unless-stopped

  awx:
    image: ansible/awx:latest
    container_name: awx
    depends_on:
      - awx-postgres
      - awx-redis
    environment:
      - AWX_ADMIN_USER=admin
      - AWX_ADMIN_PASSWORD=password
      - AWX_POSTGRES_HOST=awx-postgres
      - AWX_POSTGRES_PORT=5432
      - AWX_POSTGRES_USER=awx
      - AWX_POSTGRES_PASSWORD=awx
      - AWX_POSTGRES_DATABASE=awx
      - AWX_REDIS_HOST=awx-redis
      - AWX_REDIS_PORT=6379
    volumes:
      - "{{ awx_data_dir }}/config:/etc/tower"
      - "{{ playbooks_dir }}:/var/lib/awx/projects"
    networks:
      - netbox_awx_network
    restart: unless-stopped

  awx-postgres:
    image: postgres:12
    container_name: awx-postgres
    environment:
      - POSTGRES_USER=awx
      - POSTGRES_PASSWORD=awx
      - POSTGRES_DB=awx
    volumes:
      - "{{ awx_data_dir }}/postgres:/var/lib/postgresql/data"
    networks:
      - netbox_awx_network
    restart: unless-stopped

  awx-redis:
    image: redis:6
    container_name: awx-redis
    networks:
      - netbox_awx_network
    restart: unless-stopped

networks:
  netbox_awx_network:
    driver: bridge
```

## Шаблон динамической инвентаризации (netbox_awx_inventory.yml.j2)

```yaml
#!/usr/bin/env python3

import json
import os
import requests
from requests.packages.urllib3.exceptions import InsecureRequestWarning

# Disable SSL warnings
requests.packages.urllib3.disable_warnings(InsecureRequestWarning)

NETBOX_URL = "http://netbox/api"
NETBOX_TOKEN = "{{ netbox_api_token }}"
VERIFY_SSL = False

def fetch_netbox_data(endpoint):
    headers = {
        "Authorization": f"Token {NETBOX_TOKEN}",
        "Accept": "application/json",
    }
    url = f"{NETBOX_URL}/{endpoint}"
    response = requests.get(url, headers=headers, verify=VERIFY_SSL)
    response.raise_for_status()
    return response.json().get("results", [])

def main():
    inventory = {
        "_meta": {
            "hostvars": {}
        },
        "all": {
            "children": []
        }
    }

    # Fetch devices from NetBox
    devices = fetch_netbox_data("dcim/devices/?status=active")
    
    for device in devices:
        device_name = device["name"]
        device_ip = device.get("primary_ip", {}).get("address", "").split("/")[0]
        
        if not device_ip:
            continue
            
        # Add device to inventory
        inventory["_meta"]["hostvars"][device_name] = {
            "ansible_host": device_ip,
            "device_role": device["device_role"]["name"],
            "device_type": device["device_type"]["model"],
            "site": device["site"]["name"],
            "netbox_url": f"{NETBOX_URL}/dcim/devices/{device['id']}/"
        }
        
        # Add device to groups based on device role and site
        device_role = device["device_role"]["name"]
        site_name = device["site"]["name"]
        
        if device_role not in inventory:
            inventory[device_role] = {"hosts": []}
            inventory["all"]["children"].append(device_role)
            
        if site_name not in inventory:
            inventory[site_name] = {"hosts": []}
            inventory["all"]["children"].append(site_name)
            
        inventory[device_role]["hosts"].append(device_name)
        inventory[site_name]["hosts"].append(device_name)

    print(json.dumps(inventory, indent=2))

if __name__ == "__main__":
    main()
```

## Настройка интеграции

После развертывания необходимо выполнить следующие шаги в AWX:

1. Создать новый проект для хранения плейбуков
2. Создать инвентарь и указать скрипт динамической инвентаризации
3. Настроить credentials для доступа к NetBox API
4. Создать шаблон задания (job template) для выполнения плейбуков

## Переменные по умолчанию (defaults/main.yml)

```yaml
# Директории проекта
project_dir: "/opt/netbox_awx"
playbooks_dir: "{{ project_dir }}/playbooks"

# Настройки Nginx
nginx_config_dir: "{{ project_dir }}/nginx"

# Настройки NetBox
netbox_data_dir: "{{ project_dir }}/netbox"
netbox_api_token: "your_netbox_api_token_here"

# Настройки AWX
awx_data_dir: "{{ project_dir }}/awx"
awx_admin_user: "admin"
awx_admin_password: "password"
awx_inventory_script_path: "{{ awx_data_dir }}/netbox_inventory.py"
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
- hosts: all
  become: yes
  roles:
    - netbox-awx-docker
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory playbook.yml
```

После выполнения роли у вас будет полностью настроенное окружение с:
- Nginx в качестве reverse proxy
- NetBox с локальными конфигурациями
- AWX с доступом к плейбукам и динамической инвентаризации
- Автоматической интеграцией между AWX и NetBox