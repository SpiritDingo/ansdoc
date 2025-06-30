# Роль Ansible для установки и настройки Semaphore UI Enterprise с Docker Compose v2+

Вот полная роль Ansible для установки и настройки Semaphore UI Enterprise с использованием Docker Compose версии 2 и выше.

## Структура роли

```
roles/semaphore_ui_enterprise/
├── defaults
│   └── main.yml
├── files
│   └── semaphore-docker-compose.yml.j2
├── tasks
│   └── main.yml
└── templates
    └── semaphore.env.j2
```

## 1. defaults/main.yml

```yaml
---
# Настройки Semaphore UI Enterprise
semaphore_version: "latest"
semaphore_data_dir: "/opt/semaphore"
semaphore_port: 3000
semaphore_db_port: 5432
semaphore_db_name: "semaphore"
semaphore_db_user: "semaphore"
semaphore_db_password: "change_this_password"
semaphore_admin_password: "admin123"
semaphore_admin_email: "admin@example.com"
semaphore_admin_name: "Admin"
semaphore_encryption_key: "change_this_encryption_key"

# Настройки Docker
docker_compose_version: "v2"
```

## 2. tasks/main.yml

```yaml
---
- name: Ensure directories exist
  file:
    path: "{{ semaphore_data_dir }}/{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - config
    - data
    - postgres-data

- name: Copy docker-compose template
  template:
    src: semaphore-docker-compose.yml.j2
    dest: "{{ semaphore_data_dir }}/docker-compose.yml"
    mode: '0644'

- name: Copy environment file template
  template:
    src: semaphore.env.j2
    dest: "{{ semaphore_data_dir }}/config/semaphore.env"
    mode: '0600'

- name: Pull Semaphore UI Enterprise images
  community.docker.docker_compose:
    project_src: "{{ semaphore_data_dir }}"
    pull: yes

- name: Start Semaphore UI Enterprise services
  community.docker.docker_compose:
    project_src: "{{ semaphore_data_dir }}"
    state: present
    recreate: always
    restart: yes

- name: Wait for Semaphore to become available
  uri:
    url: "http://localhost:{{ semaphore_port }}/api/auth/login"
    method: GET
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 10
  delay: 10
```

## 3. templates/semaphore.env.j2

```ini
SEMAPHORE_DB_USER={{ semaphore_db_user }}
SEMAPHORE_DB_PASS={{ semaphore_db_password }}
SEMAPHORE_DB_HOST=postgres
SEMAPHORE_DB_PORT={{ semaphore_db_port }}
SEMAPHORE_DB={{ semaphore_db_name }}
SEMAPHORE_DB_SSL=disable
SEMAPHORE_PLAYBOOK_PATH=/tmp/semaphore
SEMAPHORE_ADMIN={{ semaphore_admin_name }}
SEMAPHORE_ADMIN_EMAIL={{ semaphore_admin_email }}
SEMAPHORE_ADMIN_PASSWORD={{ semaphore_admin_password }}
SEMAPHORE_ENCRYPTION={{ semaphore_encryption_key }}
SEMAPHORE_ACCESS_KEY_ENCRYPTION={{ semaphore_encryption_key }}
SEMAPHORE_LDAP_ENABLED=false
```

## 4. files/semaphore-docker-compose.yml.j2

```yaml
version: '3.8'

services:
  semaphore:
    image: semaphoreui/semaphore:enterprise-{{ semaphore_version }}
    container_name: semaphore
    restart: unless-stopped
    ports:
      - "{{ semaphore_port }}:3000"
    volumes:
      - "{{ semaphore_data_dir }}/data:/etc/semaphore"
      - "{{ semaphore_data_dir }}/config/semaphore.env:/etc/semaphore/semaphore.env"
    environment:
      - SEMAPHORE_DB_DIALECT=postgres
    depends_on:
      - postgres
    networks:
      - semaphore-net

  postgres:
    image: postgres:15-alpine
    container_name: semaphore-postgres
    restart: unless-stopped
    ports:
      - "{{ semaphore_db_port }}:5432"
    volumes:
      - "{{ semaphore_data_dir }}/postgres-data:/var/lib/postgresql/data"
    environment:
      POSTGRES_USER: "{{ semaphore_db_user }}"
      POSTGRES_PASSWORD: "{{ semaphore_db_password }}"
      POSTGRES_DB: "{{ semaphore_db_name }}"
    networks:
      - semaphore-net

networks:
  semaphore-net:
    driver: bridge
```

## Использование роли

1. Создайте playbook (например, `install_semaphore.yml`):

```yaml
---
- hosts: semaphore_servers
  become: yes
  roles:
    - semaphore_ui_enterprise
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini install_semaphore.yml
```

## Дополнительные настройки

Вы можете переопределить любые параметры из `defaults/main.yml` в playbook или inventory:

```yaml
- hosts: semaphore_servers
  become: yes
  vars:
    semaphore_version: "2.8.0"
    semaphore_admin_password: "my_secure_password"
  roles:
    - semaphore_ui_enterprise
```

## После установки

После успешной установки:
1. Откройте Semaphore UI в браузере: `http://<ваш_сервер>:3000`
2. Войдите с email и паролем, указанными в переменных
3. Настройте проекты, инвентари и шаблоны задач

Для обновления Semaphore просто измените переменную `semaphore_version` и перезапустите playbook.



# Обновленная роль Ansible для Semaphore UI Enterprise с Docker Compose v2

Вот исправленная роль, использующая модуль `community.docker.docker_compose_v2` вместо устаревшего `community.docker.docker_compose`.

## Обновленный tasks/main.yml

```yaml
---
- name: Ensure directories exist
  file:
    path: "{{ semaphore_data_dir }}/{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - config
    - data
    - postgres-data

- name: Copy docker-compose template
  template:
    src: semaphore-docker-compose.yml.j2
    dest: "{{ semaphore_data_dir }}/docker-compose.yml"
    mode: '0644'

- name: Copy environment file template
  template:
    src: semaphore.env.j2
    dest: "{{ semaphore_data_dir }}/config/semaphore.env"
    mode: '0600'

- name: Pull Semaphore UI Enterprise images
  community.docker.docker_compose_v2:
    project_src: "{{ semaphore_data_dir }}"
    pull: yes

- name: Start Semaphore UI Enterprise services
  community.docker.docker_compose_v2:
    project_src: "{{ semaphore_data_dir }}"
    state: present
    recreate: always
    restart: yes

- name: Wait for Semaphore to become available
  uri:
    url: "http://localhost:{{ semaphore_port }}/api/auth/login"
    method: GET
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 10
  delay: 10
```

## Требования к системе

1. Убедитесь, что у вас установлены:
   - Ansible 2.10+
   - Docker Compose v2+ (как плагин Docker)
   - Коллекция `community.docker` версии 3.0.0 или новее

2. Установите необходимые коллекции:

```bash
ansible-galaxy collection install community.docker
```

## Важные изменения

1. Замена модуля:
   - Старый: `community.docker.docker_compose`
   - Новый: `community.docker.docker_compose_v2`

2. Параметры модуля остались практически такими же, но работают с Docker Compose v2.

## Дополнительные рекомендации

1. Для проверки версии Docker Compose на целевых хостах добавьте задачу:

```yaml
- name: Verify Docker Compose version
  command: docker compose version
  register: compose_version
  changed_when: false

- name: Fail if Docker Compose v1 is detected
  fail:
    msg: "Docker Compose v1 detected. Please upgrade to Docker Compose v2+."
  when: "'Docker Compose version v2' not in compose_version.stdout"
```

2. Для автоматического обновления Docker Compose до v2 можно добавить:

```yaml
- name: Install Docker Compose v2 plugin
  become: yes
  ansible.builtin.get_url:
    url: "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-linux-{{ ansible_architecture }}"
    dest: "/usr/libexec/docker/cli-plugins/docker-compose"
    mode: '0755'
    owner: root
    group: root
  when: "'Docker Compose version v2' not in compose_version.stdout"
```

Эта обновленная роль обеспечит совместимость с современными версиями Docker и Ansible.