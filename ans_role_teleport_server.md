# Ansible роль для установки и настройки Teleport на Linux

Роль Ansible для установки и настройки сервера Teleport (Gravitational Teleport) - современного SSH-сервера с возможностями единого входа (SSO), аудита сессий и доступа к Kubernetes.

## Функционал роли

1. Установка Teleport из официальных репозиториев
2. Базовая настройка сервера (auth, proxy, node)
3. Настройка systemd сервиса
4. Генерация начального токена для добавления узлов
5. Опциональная настройка Let's Encrypt сертификатов

## Структура роли

```
roles/teleport/
├── defaults
│   └── main.yml       # переменные по умолчанию
├── files
│   └── teleport.yaml  # шаблон конфигурации
├── handlers
│   └── main.yml       # обработчики (перезапуск сервиса)
├── tasks
│   └── main.yml       # основные задачи
└── templates
    └── teleport.service.j2  # шаблон systemd сервиса
```

## Пример использования

### playbook.yml

```yaml
- hosts: teleport_servers
  become: yes
  roles:
    - role: teleport
      teleport_version: "14.3.4"
      teleport_edition: "oss"  # или "ent" для Enterprise
      teleport_auth_token: "your_secure_token"
      teleport_cluster_name: "example.com"
      teleport_acme_enabled: true
      teleport_acme_email: "admin@example.com"
```

### inventory.yml

```yaml
all:
  children:
    teleport_servers:
      hosts:
        teleport1.example.com:
          teleport_role: "auth,proxy,node"
        teleport2.example.com:
          teleport_role: "node"
```

## Основные переменные (defaults/main.yml)

```yaml
# Версия Teleport для установки
teleport_version: "14.3.4"

# Редакция (oss или ent)
teleport_edition: "oss"

# Роли сервера (auth,proxy,node)
teleport_role: "auth,proxy,node"

# Имя кластера
teleport_cluster_name: "example.com"

# Токен для присоединения узлов
teleport_auth_token: ""

# Настройки Let's Encrypt
teleport_acme_enabled: false
teleport_acme_email: ""

# Путь для данных Teleport
teleport_data_dir: "/var/lib/teleport"

# Пользователь и группа
teleport_user: "teleport"
teleport_group: "teleport"
```

## Задачи (tasks/main.yml)

```yaml
---
- name: Install dependencies
  apt:
    name:
      - curl
      - apt-transport-https
    state: present
  when: ansible_os_family == 'Debian'

- name: Add Teleport repository key
  apt_key:
    url: "https://deb.releases.teleport.dev/gpg"
    state: present
  when: ansible_os_family == 'Debian'

- name: Add Teleport repository
  apt_repository:
    repo: "deb https://deb.releases.teleport.dev/{{ 'stable' if teleport_edition == 'oss' else 'stable/v13' }} {{ ansible_distribution_release }} main"
    state: present
    update_cache: yes
  when: ansible_os_family == 'Debian'

- name: Install Teleport package
  package:
    name: "teleport={{ teleport_version }}"
    state: present

- name: Create Teleport data directory
  file:
    path: "{{ teleport_data_dir }}"
    state: directory
    owner: "{{ teleport_user }}"
    group: "{{ teleport_group }}"
    mode: '0750'

- name: Configure Teleport
  template:
    src: teleport.yaml.j2
    dest: /etc/teleport.yaml
    owner: "{{ teleport_user }}"
    group: "{{ teleport_group }}"
    mode: '0640'
  notify: restart teleport

- name: Configure systemd service
  template:
    src: teleport.service.j2
    dest: /etc/systemd/system/teleport.service
    owner: root
    group: root
    mode: '0644'
  notify:
    - daemon-reload
    - restart teleport

- name: Enable and start Teleport service
  service:
    name: teleport
    state: started
    enabled: yes
```

## Шаблон конфигурации (templates/teleport.yaml.j2)

```yaml
version: v3

teleport:
  nodename: {{ ansible_hostname }}
  data_dir: {{ teleport_data_dir }}
  auth_token: {{ teleport_auth_token }}
  log:
    output: stderr
    severity: INFO

auth_service:
  enabled: {{ "auth" in teleport_role.split(',') }}
  cluster_name: {{ teleport_cluster_name }}
  listen_addr: 0.0.0.0:3025

ssh_service:
  enabled: {{ "node" in teleport_role.split(',') }}
  labels:
    env: production
    hostname: {{ ansible_hostname }}

proxy_service:
  enabled: {{ "proxy" in teleport_role.split(',') }}
  listen_addr: 0.0.0.0:3023
  web_listen_addr: 0.0.0.0:3080
  public_addr: {{ teleport_cluster_name }}:3080
  {% if teleport_acme_enabled %}
  acme:
    enabled: true
    email: {{ teleport_acme_email }}
  {% endif %}
```

## Обработчики (handlers/main.yml)

```yaml
- name: restart teleport
  service:
    name: teleport
    state: restarted

- name: daemon-reload
  systemd:
    daemon_reload: yes
```

## Дополнительные возможности

1. **Настройка аутентификации через OIDC/GitHub/SAML** - можно добавить соответствующие секции в конфигурацию
2. **Настройка кластера с несколькими auth-серверами** - требует дополнительной конфигурации
3. **Интеграция с Kubernetes** - добавление соответствующей секции в конфигурацию

Для использования роли в production рекомендуется:
1. Установить безопасный токен через переменную `teleport_auth_token`
2. Настроить брандмауэр для защиты портов Teleport
3. Регулярно обновлять Teleport до актуальной версии