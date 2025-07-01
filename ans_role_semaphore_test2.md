# Ansible роль для развертывания Semaphore с переменными на основе шаблона

Вот пример Ansible роли для развертывания Semaphore с использованием переменных из вашего шаблона Docker Compose.

## Структура роли

```
roles/semaphore/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── docker-compose.yml.j2
└── vars
    └── main.yml
```

## Файлы роли

### `defaults/main.yml`

```yaml
---
# Настройки Semaphore
semaphore_version: "v2.15.0"
semaphore_port: 3000

# Настройки базы данных
semaphore_db_image: "postgres"
semaphore_db_user: "semaphore"
semaphore_db_password: "y4teq5ojw7"
semaphore_db_name: "semaphore"

# Настройки администратора
semaphore_admin_username: "admin"
semaphore_admin_password: "admin123!"
semaphore_admin_name: "Admin"
semaphore_admin_email: "admin@localhost"

# Настройки runner
semaphore_use_remote_runner: "True"
semaphore_runner_registration_token: "i+IYthCbXvolssK5Dzf1v6nxGDQamedkoDuJpx8Av3A="

# Настройки Ansible
ansible_host_key_checking: "True"

# Настройки сети
semaphore_network_driver: "bridge"
```

### `vars/main.yml`

```yaml
---
# Имена volumes
semaphore_volumes:
  - semaphore_data
  - semaphore_config
  - semaphore_tmp
  - semaphore_postgres
```

### `templates/docker-compose.yml.j2`

```yaml
version: '3'

services:
  semaphore_db:
    image: {{ semaphore_db_image }}
    environment:
      POSTGRES_USER: {{ semaphore_db_user }}
      POSTGRES_PASSWORD: {{ semaphore_db_password }}
      POSTGRES_DB: {{ semaphore_db_name }}
    volumes:
      - semaphore_postgres:/var/lib/postgresql/data
    networks:
      - semaphore_network

  semaphore:
    ports:
      - "{{ semaphore_port }}:3000"
    depends_on:
      - semaphore_db
    image: semaphoreui/semaphore:{{ semaphore_version }}
    environment:
      SEMAPHORE_DB_DIALECT: postgres
      SEMAPHORE_DB_HOST: semaphore_db
      SEMAPHORE_DB_NAME: {{ semaphore_db_name }}
      SEMAPHORE_DB_USER: {{ semaphore_db_user }}
      SEMAPHORE_DB_PASS: {{ semaphore_db_password }}
      SEMAPHORE_ADMIN: {{ semaphore_admin_username }}
      SEMAPHORE_ADMIN_PASSWORD: {{ semaphore_admin_password }}
      SEMAPHORE_ADMIN_NAME: {{ semaphore_admin_name }}
      SEMAPHORE_ADMIN_EMAIL: {{ semaphore_admin_email }}
      SEMAPHORE_USE_REMOTE_RUNNER: "{{ semaphore_use_remote_runner }}"
      SEMAPHORE_RUNNER_REGISTRATION_TOKEN: "{{ semaphore_runner_registration_token }}"
      ANSIBLE_HOST_KEY_CHECKING: "{{ ansible_host_key_checking }}"
    volumes:
      - semaphore_data:/var/lib/semaphore
      - semaphore_config:/etc/semaphore
      - semaphore_tmp:/tmp/semaphore
    networks:
      - semaphore_network

volumes:
  {% for volume in semaphore_volumes %}
  {{ volume }}:
  {% endfor %}

networks:
  semaphore_network:
    driver: "{{ semaphore_network_driver }}"
```

### `tasks/main.yml`

```yaml
---
- name: Ensure Docker is installed
  become: yes
  ansible.builtin.apt:
    name:
      - docker.io
      - docker-compose
    state: present
    update_cache: yes

- name: Create directory for Semaphore
  ansible.builtin.file:
    path: /opt/semaphore
    state: directory
    mode: '0755'

- name: Generate docker-compose.yml
  ansible.builtin.template:
    src: docker-compose.yml.j2
    dest: /opt/semaphore/docker-compose.yml
    mode: '0644'

- name: Start Semaphore services
  ansible.builtin.command: docker-compose up -d
  args:
    chdir: /opt/semaphore
  register: docker_compose_up
  changed_when: "'Creating' in docker_compose_up.stdout"
```

## Использование роли

1. Создайте playbook (например, `deploy_semaphore.yml`):

```yaml
---
- hosts: semaphore_servers
  become: yes
  roles:
    - semaphore
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory deploy_semaphore.yml
```

## Кастомизация

Вы можете переопределить любые переменные из `defaults/main.yml` при вызове роли:

```yaml
- hosts: semaphore_servers
  become: yes
  vars:
    semaphore_admin_password: "new_secure_password"
    semaphore_db_password: "another_secure_password"
  roles:
    - semaphore
```

Или через файл переменных, inventory или командную строку.



# Ansible роль для развертывания Semaphore с volumes в /opt/semaphore

Вот обновленная версия роли, где все volumes монтируются в `/opt/semaphore/`.

## Обновленная структура роли

```
roles/semaphore/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   └── docker-compose.yml.j2
└── vars
    └── main.yml
```

## Обновленные файлы роли

### `defaults/main.yml` (добавлены пути к volumes)

```yaml
---
# Настройки Semaphore
semaphore_version: "v2.15.0"
semaphore_port: 3000

# Настройки базы данных
semaphore_db_image: "postgres"
semaphore_db_user: "semaphore"
semaphore_db_password: "y4teq5ojw7"
semaphore_db_name: "semaphore"

# Настройки администратора
semaphore_admin_username: "admin"
semaphore_admin_password: "admin123!"
semaphore_admin_name: "Admin"
semaphore_admin_email: "admin@localhost"

# Настройки runner
semaphore_use_remote_runner: "True"
semaphore_runner_registration_token: "i+IYthCbXvolssK5Dzf1v6nxGDQamedkoDuJpx8Av3A="

# Настройки Ansible
ansible_host_key_checking: "True"

# Настройки сети
semaphore_network_driver: "bridge"

# Пути к volumes
semaphore_base_dir: "/opt/semaphore"
semaphore_data_dir: "{{ semaphore_base_dir }}/data"
semaphore_config_dir: "{{ semaphore_base_dir }}/config"
semaphore_tmp_dir: "{{ semaphore_base_dir }}/tmp"
semaphore_postgres_dir: "{{ semaphore_base_dir }}/postgres"
```

### `vars/main.yml` (без изменений)

```yaml
---
# Имена volumes
semaphore_volumes:
  - semaphore_data
  - semaphore_config
  - semaphore_tmp
  - semaphore_postgres
```

### `templates/docker-compose.yml.j2` (обновлены пути volumes)

```yaml
version: '3'

services:
  semaphore_db:
    image: {{ semaphore_db_image }}
    environment:
      POSTGRES_USER: {{ semaphore_db_user }}
      POSTGRES_PASSWORD: {{ semaphore_db_password }}
      POSTGRES_DB: {{ semaphore_db_name }}
    volumes:
      - {{ semaphore_postgres_dir }}:/var/lib/postgresql/data
    networks:
      - semaphore_network

  semaphore:
    ports:
      - "{{ semaphore_port }}:3000"
    depends_on:
      - semaphore_db
    image: semaphoreui/semaphore:{{ semaphore_version }}
    environment:
      SEMAPHORE_DB_DIALECT: postgres
      SEMAPHORE_DB_HOST: semaphore_db
      SEMAPHORE_DB_NAME: {{ semaphore_db_name }}
      SEMAPHORE_DB_USER: {{ semaphore_db_user }}
      SEMAPHORE_DB_PASS: {{ semaphore_db_password }}
      SEMAPHORE_ADMIN: {{ semaphore_admin_username }}
      SEMAPHORE_ADMIN_PASSWORD: {{ semaphore_admin_password }}
      SEMAPHORE_ADMIN_NAME: {{ semaphore_admin_name }}
      SEMAPHORE_ADMIN_EMAIL: {{ semaphore_admin_email }}
      SEMAPHORE_USE_REMOTE_RUNNER: "{{ semaphore_use_remote_runner }}"
      SEMAPHORE_RUNNER_REGISTRATION_TOKEN: "{{ semaphore_runner_registration_token }}"
      ANSIBLE_HOST_KEY_CHECKING: "{{ ansible_host_key_checking }}"
    volumes:
      - {{ semaphore_data_dir }}:/var/lib/semaphore
      - {{ semaphore_config_dir }}:/etc/semaphore
      - {{ semaphore_tmp_dir }}:/tmp/semaphore
    networks:
      - semaphore_network

volumes:
  {% for volume in semaphore_volumes %}
  {{ volume }}:
  {% endfor %}

networks:
  semaphore_network:
    driver: "{{ semaphore_network_driver }}"
```

### `tasks/main.yml` (добавлено создание директорий)

```yaml
---
- name: Ensure Docker is installed
  become: yes
  ansible.builtin.apt:
    name:
      - docker.io
      - docker-compose
    state: present
    update_cache: yes

- name: Create base directory for Semaphore
  ansible.builtin.file:
    path: "{{ semaphore_base_dir }}"
    state: directory
    mode: '0755'

- name: Create data directories
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ semaphore_data_dir }}"
    - "{{ semaphore_config_dir }}"
    - "{{ semaphore_tmp_dir }}"
    - "{{ semaphore_postgres_dir }}"

- name: Set permissions for data directories
  ansible.builtin.file:
    path: "{{ item }}"
    mode: '0777'  # Docker требует правильные permissions
    recurse: yes
  loop:
    - "{{ semaphore_data_dir }}"
    - "{{ semaphore_config_dir }}"
    - "{{ semaphore_tmp_dir }}"
    - "{{ semaphore_postgres_dir }}"

- name: Generate docker-compose.yml
  ansible.builtin.template:
    src: docker-compose.yml.j2
    dest: "{{ semaphore_base_dir }}/docker-compose.yml"
    mode: '0644'

- name: Start Semaphore services
  ansible.builtin.command: docker-compose up -d
  args:
    chdir: "{{ semaphore_base_dir }}"
  register: docker_compose_up
  changed_when: "'Creating' in docker_compose_up.stdout"
```

## Изменения в роли

1. Добавлены переменные для путей к директориям в `defaults/main.yml`:
   - `semaphore_base_dir` - базовая директория
   - `semaphore_data_dir` - для данных Semaphore
   - `semaphore_config_dir` - для конфигурации
   - `semaphore_tmp_dir` - для временных файлов
   - `semaphore_postgres_dir` - для данных PostgreSQL

2. Обновлен шаблон `docker-compose.yml.j2` с использованием новых переменных путей

3. В tasks добавлены:
   - Создание всех необходимых поддиректорий
   - Установка правильных permissions (0777 для работы Docker)

## Преимущества такого подхода

1. Все данные хранятся в одной предсказуемой директории (`/opt/semaphore/`)
2. Упрощается резервное копирование (можно бэкапить всю `/opt/semaphore`)
3. Упрощается миграция на другой сервер
4. Более прозрачная структура хранения данных

Для применения роли используйте тот же playbook, что и в предыдущем варианте.