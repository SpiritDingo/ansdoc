# Роль Ansible для установки и настройки NetBox

Ниже представлена базовая структура роли Ansible для установки и настройки NetBox - популярного инструмента для управления IP-адресами и центрами данных.

## Структура роли

```
netbox/
├── defaults/
│   └── main.yml          # Значения переменных по умолчанию
├── files/
│   ├── netbox.service    # Файл systemd unit для NetBox
│   └── nginx.conf        # Конфигурация Nginx для NetBox
├── handlers/
│   └── main.yml          # Обработчики для перезагрузки служб
├── tasks/
│   └── main.yml          # Основные задачи установки
├── templates/
│   └── configuration.py.j2  # Шаблон конфигурации NetBox
└── vars/
    └── main.yml          # Основные переменные роли
```

## Основные файлы роли

### defaults/main.yml

```yaml
---
# Версия NetBox для установки
netbox_version: "v3.6.2"

# Пользователь и группа для NetBox
netbox_user: "netbox"
netbox_group: "netbox"

# Директории установки
netbox_home: "/opt/netbox"
netbox_static_root: "{{ netbox_home }}/static"
netbox_media_root: "{{ netbox_home }}/media"

# Настройки базы данных
netbox_db_name: "netbox"
netbox_db_user: "netbox"
netbox_db_password: "changeme"
netbox_db_host: "localhost"
netbox_db_port: "5432"

# Настройки Redis
netbox_redis_host: "localhost"
netbox_redis_port: "6379"
netbox_redis_password: ""
netbox_redis_db: "0"

# Настройки администратора
netbox_admin_name: "Admin"
netbox_admin_email: "admin@example.com"
netbox_admin_password: "admin"

# Настройки сервера
netbox_server_name: "netbox.example.com"
netbox_server_port: "8000"
netbox_secret_key: "generate-me-with-ansible-vault"
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  apt:
    name:
      - python3
      - python3-pip
      - python3-venv
      - python3-dev
      - build-essential
      - libxml2-dev
      - libxslt1-dev
      - libffi-dev
      - libpq-dev
      - libssl-dev
      - zlib1g-dev
      - postgresql
      - postgresql-contrib
      - redis-server
      - nginx
    state: present
    update_cache: yes

- name: Создание пользователя и группы для NetBox
  user:
    name: "{{ netbox_user }}"
    group: "{{ netbox_group }}"
    system: yes
    shell: "/bin/bash"
    home: "{{ netbox_home }}"

- name: Создание директорий для NetBox
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ netbox_user }}"
    group: "{{ netbox_group }}"
    mode: '0755'
  loop:
    - "{{ netbox_home }}"
    - "{{ netbox_static_root }}"
    - "{{ netbox_media_root }}"

- name: Клонирование репозитория NetBox
  git:
    repo: "https://github.com/netbox-community/netbox.git"
    dest: "{{ netbox_home }}"
    version: "{{ netbox_version }}"
    depth: 1

- name: Настройка виртуального окружения Python
  pip:
    requirements: "{{ netbox_home }}/requirements.txt"
    virtualenv: "{{ netbox_home }}/venv"
    virtualenv_python: python3
    executable: pip3

- name: Копирование конфигурационного файла NetBox
  template:
    src: "configuration.py.j2"
    dest: "{{ netbox_home }}/netbox/netbox/configuration.py"
    owner: "{{ netbox_user }}"
    group: "{{ netbox_group }}"
    mode: '0644'

- name: Настройка базы данных PostgreSQL
  postgresql_db:
    name: "{{ netbox_db_name }}"
    encoding: "UTF8"
    lc_collate: "en_US.UTF-8"
    lc_ctype: "en_US.UTF-8"
    template: "template0"

- name: Создание пользователя базы данных
  postgresql_user:
    name: "{{ netbox_db_user }}"
    password: "{{ netbox_db_password }}"
    db: "{{ netbox_db_name }}"
    priv: "ALL"

- name: Применение миграций базы данных
  command: "{{ netbox_home }}/venv/bin/python3 {{ netbox_home }}/netbox/manage.py migrate"
  become_user: "{{ netbox_user }}"

- name: Создание суперпользователя
  command: >
    {{ netbox_home }}/venv/bin/python3 {{ netbox_home }}/netbox/manage.py createsuperuser
    --noinput --username admin --email {{ netbox_admin_email }}
  become_user: "{{ netbox_user }}"
  when: netbox_create_superuser

- name: Сбор статических файлов
  command: "{{ netbox_home }}/venv/bin/python3 {{ netbox_home }}/netbox/manage.py collectstatic --no-input"
  become_user: "{{ netbox_user }}"

- name: Копирование systemd unit файла
  copy:
    src: "netbox.service"
    dest: "/etc/systemd/system/netbox.service"
    owner: root
    group: root
    mode: '0644'
  notify: reload systemd

- name: Копирование конфигурации Nginx
  copy:
    src: "nginx.conf"
    dest: "/etc/nginx/sites-available/netbox"
    owner: root
    group: root
    mode: '0644'

- name: Активация сайта Nginx
  file:
    src: "/etc/nginx/sites-available/netbox"
    dest: "/etc/nginx/sites-enabled/netbox"
    state: link
  notify: restart nginx

- name: Запуск и включение служб
  service:
    name: "{{ item }}"
    state: started
    enabled: yes
  loop:
    - netbox
    - nginx
    - redis-server
    - postgresql
```

### handlers/main.yml

```yaml
---
- name: reload systemd
  systemd:
    daemon_reload: yes

- name: restart netbox
  service:
    name: netbox
    state: restarted

- name: restart nginx
  service:
    name: nginx
    state: restarted
```

### templates/configuration.py.j2

```python
ALLOWED_HOSTS = ['{{ netbox_server_name }}']

DATABASE = {
    'NAME': '{{ netbox_db_name }}',
    'USER': '{{ netbox_db_user }}',
    'PASSWORD': '{{ netbox_db_password }}',
    'HOST': '{{ netbox_db_host }}',
    'PORT': '{{ netbox_db_port }}',
    'CONN_MAX_AGE': 300,
}

REDIS = {
    'tasks': {
        'HOST': '{{ netbox_redis_host }}',
        'PORT': {{ netbox_redis_port }},
        'PASSWORD': '{{ netbox_redis_password }}',
        'DATABASE': {{ netbox_redis_db }},
        'SSL': False,
    },
    'caching': {
        'HOST': '{{ netbox_redis_host }}',
        'PORT': {{ netbox_redis_port }},
        'PASSWORD': '{{ netbox_redis_password }}',
        'DATABASE': {{ netbox_redis_db|int + 1 }},
        'SSL': False,
    }
}

SECRET_KEY = '{{ netbox_secret_key }}'
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
---
- hosts: netbox_server
  become: yes
  roles:
    - netbox
```

2. Создайте файл с переменными для переопределения значений по умолчанию:

```yaml
---
netbox_db_password: "secure_password"
netbox_secret_key: "very_secret_key_generated_with_openssl"
netbox_server_name: "netbox.mycompany.com"
netbox_admin_email: "admin@mycompany.com"
```

3. Запустите playbook:

```bash
ansible-playbook -i inventory.ini netbox_playbook.yml --extra-vars "@vars.yml"
```

Для защиты чувствительных данных рекомендуется использовать Ansible Vault для шифрования паролей и секретных ключей.