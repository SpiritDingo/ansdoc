# Ansible роль: Установка Semaphore через Docker Compose с подключением инвентаря

## Создание структуры роли

```bash
mkdir -p roles/semaphore_docker/{tasks,handlers,templates,vars,files,defaults,meta}
cd roles/semaphore_docker
```

## Основные файлы роли

### 1. defaults/main.yml - переменные по умолчанию

```yaml
---
# Настройки Docker Compose
semaphore_compose_dir: "/opt/semaphore"
semaphore_compose_file: "docker-compose.yml"

# Настройки Semaphore
semaphore_version: "2.8.90"
semaphore_port: 3000
semaphore_admin_email: "admin@example.com"
semaphore_admin_username: "admin"
semaphore_admin_password: "admin"  # Изменить в продакшене!

# Настройки базы данных
semaphore_db_type: "postgres"  # или "mysql", "sqlite"
semaphore_db_host: "semaphore-db"
semaphore_db_port: 5432
semaphore_db_name: "semaphore"
semaphore_db_user: "semaphore"
semaphore_db_password: "semaphore_db_password"  # Изменить в продакшене!

# Настройки инвентаря
semaphore_inventory_dir: "{{ semaphore_compose_dir }}/inventory"
semaphore_ansible_config: "{{ semaphore_compose_dir }}/ansible.cfg"

# Настройки Docker
docker_compose_version: "1.29.2"
```

### 2. tasks/main.yml - основные задачи

```yaml
---
- name: Установка зависимостей
  package:
    name:
      - docker.io
      - docker-compose-plugin
      - git
      - python3-pip
    state: present

- name: Установка docker-compose через pip (если нужно)
  pip:
    name: docker-compose
    version: "{{ docker_compose_version }}"
  when: ansible_pkg_mgr == 'apt'

- name: Добавление пользователя в группу docker
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: yes

- name: Создание директорий
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ semaphore_compose_dir }}"
    - "{{ semaphore_inventory_dir }}"

- name: Копирование docker-compose файла
  template:
    src: "docker-compose.yml.j2"
    dest: "{{ semaphore_compose_dir }}/{{ semaphore_compose_file }}"
    mode: '0644'

- name: Копирование конфига Ansible
  template:
    src: "ansible.cfg.j2"
    dest: "{{ semaphore_ansible_config }}"
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
    mode: '0644'

- name: Запуск Semaphore через Docker Compose
  community.docker.docker_compose:
    project_src: "{{ semaphore_compose_dir }}"
    state: present
    restarted: yes
    recreate: always
    pull: yes

- name: Ожидание запуска Semaphore
  uri:
    url: "http://localhost:{{ semaphore_port }}/api/auth/login"
    method: GET
    status_code: 404  # ожидаем 404, так как это GET запрос к login endpoint
    timeout: 30
  register: result
  until: result.status == 404
  retries: 10
  delay: 5
```

### 3. templates/docker-compose.yml.j2 - файл Docker Compose

```yaml
version: '3.8'

services:
  semaphore:
    image: semaphoreui/semaphore:{{ semaphore_version }}
    container_name: semaphore
    restart: unless-stopped
    ports:
      - "{{ semaphore_port }}:3000"
    volumes:
      - "{{ semaphore_inventory_dir }}:/tmp/inventory"
      - "{{ semaphore_ansible_config }}:/etc/ansible/ansible.cfg"
      - "/var/run/docker.sock:/var/run/docker.sock"
    environment:
      - SEMAPHORE_DB_HOST={{ semaphore_db_host }}
      - SEMAPHORE_DB_PORT={{ semaphore_db_port }}
      - SEMAPHORE_DB_NAME={{ semaphore_db_name }}
      - SEMAPHORE_DB_USER={{ semaphore_db_user }}
      - SEMAPHORE_DB_PASS={{ semaphore_db_password }}
      - SEMAPHORE_DB_DIALECT={{ semaphore_db_type }}
      - SEMAPHORE_ADMIN={{ semaphore_admin_username }}
      - SEMAPHORE_ADMIN_EMAIL={{ semaphore_admin_email }}
      - SEMAPHORE_ADMIN_PASSWORD={{ semaphore_admin_password }}
      - SEMAPHORE_ADMIN_NAME=Semaphore Admin
      - SEMAPHORE_ACCESS_KEY_ENCRYPTION={{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}
      - SEMAPHORE_COOKIE_HASH={{ lookup('password', '/dev/null length=64 chars=ascii_letters,digits') }}
      - SEMAPHORE_COOKIE_ENCRYPTION={{ lookup('password', '/dev/null length=32 chars=ascii_letters,digits') }}
    depends_on:
      - db

  db:
    image: {{ 'postgres:13-alpine' if semaphore_db_type == 'postgres' else 'mysql:5.7' }}
    container_name: semaphore-db
    restart: unless-stopped
    environment:
      - {{ 'POSTGRES_PASSWORD' if semaphore_db_type == 'postgres' else 'MYSQL_ROOT_PASSWORD' }}={{ semaphore_db_password }}
      - {{ 'POSTGRES_USER' if semaphore_db_type == 'postgres' else 'MYSQL_USER' }}={{ semaphore_db_user }}
      - {{ 'POSTGRES_DB' if semaphore_db_type == 'postgres' else 'MYSQL_DATABASE' }}={{ semaphore_db_name }}
    volumes:
      - "{{ semaphore_compose_dir }}/db_data:/var/lib/{{ 'postgresql' if semaphore_db_type == 'postgres' else 'mysql' }}/data"
```

### 4. templates/ansible.cfg.j2 - конфиг Ansible

```ini
[defaults]
inventory = /tmp/inventory/hosts
host_key_checking = False
roles_path = /etc/ansible/roles
retry_files_enabled = False

[ssh_connection]
pipelining = True
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

## Использование роли

1. Создайте playbook `deploy_semaphore.yml`:

```yaml
---
- hosts: semaphore_servers
  become: yes
  vars:
    semaphore_db_password: "secure_db_password"
    semaphore_admin_password: "strong_admin_password"
    semaphore_inventory_dir: "/path/to/your/existing/inventory"  # Для подключения существующего инвентаря
  roles:
    - semaphore_docker
```

2. Inventory файл:

```ini
[semaphore_servers]
semaphore.example.com
```

3. Запуск playbook:

```bash
ansible-playbook -i hosts deploy_semaphore.yml
```

## Подключение существующего каталога инвентаря

Для подключения существующего каталога инвентаря:

1. Добавьте в `defaults/main.yml`:

```yaml
semaphore_use_existing_inventory: false
semaphore_existing_inventory_path: "/path/to/existing/inventory"
```

2. Обновите задачу в `tasks/main.yml`:

```yaml
- name: Подключение существующего каталога инвентаря
  file:
    src: "{{ semaphore_existing_inventory_path }}"
    dest: "{{ semaphore_inventory_dir }}"
    state: link
  when: semaphore_use_existing_inventory

- name: Создание каталога инвентаря (если не используется существующий)
  file:
    path: "{{ semaphore_inventory_dir }}"
    state: directory
    mode: '0755'
  when: not semaphore_use_existing_inventory
```

3. Используйте в playbook:

```yaml
- hosts: semaphore_servers
  become: yes
  vars:
    semaphore_use_existing_inventory: true
    semaphore_existing_inventory_path: "/ansible/inventory/production"
  roles:
    - semaphore_docker
```

## Настройка после установки

1. Откройте веб-интерфейс: `http://your-server:3000`
2. Войдите с указанными учетными данными администратора
3. В настройках проекта укажите:
   - Путь к инвентарю: `/tmp/inventory` (как смонтировано в контейнере)
   - Путь к конфигу Ansible: `/etc/ansible/ansible.cfg`
4. Настройте доступ к SSH ключам (при необходимости)

## Дополнительные улучшения

1. **Настройка HTTPS**:
Добавьте nginx или traefik перед Semaphore с SSL терминацией.

2. **Резервное копирование**:
Добавьте задачи для резервного копирования базы данных и конфигураций.

3. **Интеграция с Git**:
Настройте автоматическое обновление инвентаря из Git репозитория.