# Установка и настройка Semaphore с помощью Ansible роли

## Создание структуры роли для Semaphore

```bash
mkdir -p roles/semaphore/{tasks,handlers,templates,vars,files,defaults,meta}
cd roles/semaphore
```

## Основные файлы роли

### 1. defaults/main.yml - переменные по умолчанию

```yaml
---
# Версия Semaphore
semaphore_version: "2.8.90"

# Пользователь и группа
semaphore_user: "semaphore"
semaphore_group: "semaphore"

# Директории
semaphore_home: "/opt/semaphore"
semaphore_config_dir: "/etc/semaphore"
semaphore_log_dir: "/var/log/semaphore"

# Настройки базы данных
semaphore_db_type: "sqlite"  # или "mysql", "postgres"
semaphore_db_path: "{{ semaphore_home }}/semaphore.db"  # для sqlite
semaphore_db_host: "localhost"
semaphore_db_port: "3306"  # для mysql
semaphore_db_name: "semaphore"
semaphore_db_user: "semaphore"
semaphore_db_password: "semaphore_password"  # изменить в продакшене!

# Настройки сервера
semaphore_host: "0.0.0.0"
semaphore_port: 3000
semaphore_url: "http://localhost:{{ semaphore_port }}"

# Настройки администратора
semaphore_admin: "admin"
semaphore_admin_password: "admin"  # изменить в продакшене!
semaphore_admin_email: "admin@example.com"

# Настройки инвентаря
semaphore_inventory_dir: "{{ semaphore_home }}/inventory"
semaphore_ansible_config: "{{ semaphore_home }}/ansible.cfg"
```

### 2. tasks/main.yml - основные задачи

```yaml
---
- name: Установка зависимостей
  package:
    name:
      - git
      - ansible
      - sqlite3
    state: present

- name: Создание пользователя и группы Semaphore
  user:
    name: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
    home: "{{ semaphore_home }}"
    system: yes
    create_home: yes

- name: Создание директорий
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
    mode: '0755'
  loop:
    - "{{ semaphore_home }}"
    - "{{ semaphore_config_dir }}"
    - "{{ semaphore_log_dir }}"
    - "{{ semaphore_inventory_dir }}"

- name: Загрузка и установка Semaphore
  get_url:
    url: "https://github.com/ansible-semaphore/semaphore/releases/download/v{{ semaphore_version }}/semaphore_{{ semaphore_version }}_linux_amd64.tar.gz"
    dest: "/tmp/semaphore.tar.gz"
    mode: '0644'

- name: Распаковка Semaphore
  unarchive:
    src: "/tmp/semaphore.tar.gz"
    dest: "{{ semaphore_home }}"
    remote_src: yes
    owner: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
    mode: '0755'

- name: Создание конфигурационного файла
  template:
    src: "config.json.j2"
    dest: "{{ semaphore_config_dir }}/config.json"
    owner: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
    mode: '0640'

- name: Создание systemd unit файла
  template:
    src: "semaphore.service.j2"
    dest: "/etc/systemd/system/semaphore.service"
    mode: '0644'

- name: Создание базовой конфигурации Ansible
  template:
    src: "ansible.cfg.j2"
    dest: "{{ semaphore_ansible_config }}"
    owner: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
    mode: '0644'

- name: Создание примера инвентаря
  copy:
    content: |
      [web]
      web1.example.com
      web2.example.com

      [db]
      db1.example.com
    dest: "{{ semaphore_inventory_dir }}/hosts"
    owner: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
    mode: '0644'

- name: Обновление systemd
  systemd:
    daemon_reload: yes

- name: Запуск и включение службы Semaphore
  service:
    name: semaphore
    state: started
    enabled: yes

- name: Ожидание запуска Semaphore
  uri:
    url: "{{ semaphore_url }}/api/auth/login"
    method: GET
    status_code: 404  # ожидаем 404, так как это GET запрос к login endpoint
    timeout: 30
  register: result
  until: result.status == 404
  retries: 10
  delay: 5
```

### 3. templates/config.json.j2 - основной конфиг Semaphore

```json
{
  "mysql": {
    "host": "{{ semaphore_db_host if semaphore_db_type == 'mysql' else '' }}",
    "user": "{{ semaphore_db_user if semaphore_db_type == 'mysql' else '' }}",
    "pass": "{{ semaphore_db_password if semaphore_db_type == 'mysql' else '' }}",
    "name": "{{ semaphore_db_name if semaphore_db_type == 'mysql' else '' }}",
    "port": "{{ semaphore_db_port if semaphore_db_type == 'mysql' else '' }}"
  },
  "postgres": {
    "host": "{{ semaphore_db_host if semaphore_db_type == 'postgres' else '' }}",
    "user": "{{ semaphore_db_user if semaphore_db_type == 'postgres' else '' }}",
    "pass": "{{ semaphore_db_password if semaphore_db_type == 'postgres' else '' }}",
    "name": "{{ semaphore_db_name if semaphore_db_type == 'postgres' else '' }}",
    "port": "{{ semaphore_db_port if semaphore_db_type == 'postgres' else '' }}"
  },
  "bolt": {
    "path": "{{ semaphore_db_path if semaphore_db_type == 'sqlite' else '' }}"
  },
  "ldap": {
    "enable": false
  },
  "telegram": {
    "enable": false
  },
  "slack": {
    "enable": false
  },
  "email": {
    "enable": false
  },
  "port": "{{ semaphore_port }}",
  "interface": "{{ semaphore_host }}",
  "tmp_path": "{{ semaphore_home }}/tmp",
  "cookie_hash": "{{ lookup('password', '/dev/null length=64 chars=ascii_letters,digits') }}",
  "cookie_encryption": "{{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}",
  "access_key_encryption": "{{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}",
  "email_alert": false,
  "ssh_config_path": "",
  "demo_mode": false,
  "max_parallel_tasks": 10,
  "config_path": "{{ semaphore_config_dir }}"
}
```

### 4. templates/semaphore.service.j2 - unit файл для systemd

```ini
[Unit]
Description=Semaphore Ansible UI
After=network.target

[Service]
Type=simple
User={{ semaphore_user }}
Group={{ semaphore_group }}
WorkingDirectory={{ semaphore_home }}
ExecStart={{ semaphore_home }}/semaphore -config {{ semaphore_config_dir }}/config.json
Restart=always
RestartSec=5
Environment=HOME={{ semaphore_home }}

[Install]
WantedBy=multi-user.target
```

### 5. templates/ansible.cfg.j2 - конфиг Ansible для Semaphore

```ini
[defaults]
inventory = {{ semaphore_inventory_dir }}/hosts
host_key_checking = False
roles_path = {{ semaphore_home }}/roles
retry_files_enabled = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

## Использование роли

1. Создайте playbook `semaphore.yml`:

```yaml
---
- hosts: semaphore_servers
  become: yes
  vars:
    semaphore_db_type: "mysql"
    semaphore_db_host: "db.example.com"
    semaphore_db_password: "secure_password_here"
    semaphore_admin_password: "strong_admin_password"
  roles:
    - semaphore
```

2. Inventory файл (`hosts` или другой):

```ini
[semaphore_servers]
semaphore.example.com

[db_servers]
db.example.com
```

3. Запуск playbook:

```bash
ansible-playbook -i hosts semaphore.yml
```

## Настройка подключения к существующему каталогу инвентаря

Если вам нужно подключить существующий каталог инвентаря:

1. Добавьте в `defaults/main.yml`:

```yaml
semaphore_existing_inventory: false
semaphore_existing_inventory_path: "/path/to/existing/inventory"
```

2. Обновите задачу создания инвентаря в `tasks/main.yml`:

```yaml
- name: Подключение существующего каталога инвентаря
  file:
    src: "{{ semaphore_existing_inventory_path }}"
    dest: "{{ semaphore_inventory_dir }}"
    state: link
    owner: "{{ semaphore_user }}"
    group: "{{ semaphore_group }}"
  when: semaphore_existing_inventory
```

3. Используйте в playbook:

```yaml
- hosts: semaphore_servers
  become: yes
  vars:
    semaphore_existing_inventory: true
    semaphore_existing_inventory_path: "/ansible/inventory"
  roles:
    - semaphore
```

## Настройка после установки

После установки откройте веб-интерфейс Semaphore по адресу `http://your-server:3000` и выполните первоначальную настройку:

1. Войдите с учетными данными администратора (указанными в переменных)
2. Перейдите в настройки проекта
3. Укажите путь к инвентарю (`{{ semaphore_inventory_dir }}`)
4. Настройте доступ к SSH ключам (при необходимости)
5. Создайте первый проект и настройте окружения