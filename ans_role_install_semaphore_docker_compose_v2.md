# Ansible роль для развертывания Semaphore через Docker Compose с полной интеграцией Ansible

## Полная структура решения

```
ansible-semaphore-deploy/
├── inventories/
│   ├── production/
│   │   ├── hosts
│   │   └── group_vars/
│   └── staging/
│       ├── hosts
│       └── group_vars/
├── roles/
│   └── semaphore_docker/
│       ├── defaults/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       ├── templates/
│       │   ├── ansible.cfg.j2
│       │   ├── docker-compose.yml.j2
│       │   └── semaphore.env.j2
│       └── vars/
│           └── main.yml
└── playbooks/
    ├── deploy_semaphore.yml
    ├── configure_inventory.yml
    └── backup_semaphore.yml
```

## 1. Роль semaphore_docker

### defaults/main.yml

```yaml
---
# Версии
semaphore_version: "2.8.90"
docker_compose_version: "2.24.5"

# Настройки Semaphore
semaphore_port: 3000
semaphore_host: "0.0.0.0"
semaphore_admin_email: "admin@example.com"
semaphore_admin_username: "admin"
semaphore_admin_password: "changeme123"

# Настройки базы данных
semaphore_db_type: "postgres"
semaphore_db_host: "semaphore-db"
semaphore_db_port: 5432
semaphore_db_name: "semaphore"
semaphore_db_user: "semaphore"
semaphore_db_password: "semaphore_db_pass"

# Пути
semaphore_base_dir: "/opt/semaphore"
semaphore_inventory_dir: "{{ semaphore_base_dir }}/inventory"
semaphore_playbooks_dir: "{{ semaphore_base_dir }}/playbooks"
semaphore_roles_dir: "{{ semaphore_base_dir }}/roles"
semaphore_ansible_config: "{{ semaphore_base_dir }}/ansible.cfg"
```

### tasks/main.yml

```yaml
---
- name: Установка зависимостей
  package:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
      - python3-pip
    state: present

- name: Установка docker-compose через pip
  pip:
    name: docker-compose
    version: "{{ docker_compose_version }}"

- name: Создание структуры каталогов
  file:
    path: "{{ item }}"
    state: directory
    mode: '0755'
  loop:
    - "{{ semaphore_base_dir }}"
    - "{{ semaphore_inventory_dir }}"
    - "{{ semaphore_playbooks_dir }}"
    - "{{ semaphore_roles_dir }}"

- name: Развертывание docker-compose
  template:
    src: "docker-compose.yml.j2"
    dest: "{{ semaphore_base_dir }}/docker-compose.yml"
    mode: '0644'

- name: Развертывание конфигурации Ansible
  template:
    src: "ansible.cfg.j2"
    dest: "{{ semaphore_ansible_config }}"
    mode: '0644'

- name: Развертывание переменных окружения
  template:
    src: "semaphore.env.j2"
    dest: "{{ semaphore_base_dir }}/semaphore.env"
    mode: '0640'

- name: Копирование примеров плейбуков
  copy:
    src: "{{ playbook_samples }}"
    dest: "{{ semaphore_playbooks_dir }}/"
  when: playbook_samples is defined

- name: Запуск Semaphore
  community.docker.docker_compose:
    project_src: "{{ semaphore_base_dir }}"
    state: present
    restarted: yes
    pull: yes

- name: Ожидание запуска Semaphore
  uri:
    url: "http://localhost:{{ semaphore_port }}/api/auth/login"
    method: GET
    status_code: 404
    timeout: 30
  register: result
  until: result.status == 404
  retries: 10
  delay: 5
```

### templates/docker-compose.yml.j2

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
      - "{{ semaphore_playbooks_dir }}:/tmp/playbooks"
      - "{{ semaphore_roles_dir }}:/tmp/roles"
      - "{{ semaphore_ansible_config }}:/etc/ansible/ansible.cfg"
      - "/var/run/docker.sock:/var/run/docker.sock"
    env_file:
      - semaphore.env
    depends_on:
      - db

  db:
    image: postgres:13-alpine
    container_name: semaphore-db
    restart: unless-stopped
    environment:
      POSTGRES_PASSWORD: "{{ semaphore_db_password }}"
      POSTGRES_USER: "{{ semaphore_db_user }}"
      POSTGRES_DB: "{{ semaphore_db_name }}"
    volumes:
      - "./db_data:/var/lib/postgresql/data"
```

## 2. Плейбуки

### playbooks/deploy_semaphore.yml

```yaml
---
- name: Развертывание Semaphore
  hosts: semaphore_servers
  become: yes
  vars_files:
    - ../inventories/{{ inventory }}/group_vars/all.yml
  roles:
    - role: semaphore_docker
      vars:
        semaphore_admin_password: "{{ semaphore_admin_password }}"
        semaphore_db_password: "{{ semaphore_db_password }}"
        playbook_samples: "../sample_playbooks/"
```

### playbooks/configure_inventory.yml

```yaml
---
- name: Настройка инвентаря Semaphore
  hosts: semaphore_servers
  become: yes
  tasks:
    - name: Синхронизация инвентаря
      synchronize:
        src: "../inventories/{{ inventory }}/hosts"
        dest: "{{ semaphore_inventory_dir }}/"
        mode: push

    - name: Перезагрузка Semaphore
      community.docker.docker_compose:
        project_src: "{{ semaphore_base_dir }}"
        state: restarted
```

## 3. Инвентарь

### inventories/production/hosts

```ini
[semaphore_servers]
semaphore-prod.example.com

[web_servers]
web1.example.com
web2.example.com

[db_servers]
db1.example.com
```

### inventories/production/group_vars/all.yml

```yaml
---
semaphore_admin_password: "ProdSecurePass123!"
semaphore_db_password: "ProdDBPass456!"
```

## 4. Пример использования

1. Клонируем репозиторий с плейбуками:
```bash
git clone https://example.com/ansible-semaphore-deploy.git
cd ansible-semaphore-deploy
```

2. Развертываем Semaphore для production:
```bash
ansible-playbook -i inventories/production/ playbooks/deploy_semaphore.yml -e "inventory=production"
```

3. Обновляем инвентарь:
```bash
ansible-playbook -i inventories/production/ playbooks/configure_inventory.yml -e "inventory=production"
```

## 5. Интеграция с существующей структурой Ansible

Для подключения существующих плейбуков и ролей:

1. Создайте символические ссылки:
```bash
ln -s /path/to/your/ansible/playbooks /opt/semaphore/playbooks
ln -s /path/to/your/ansible/roles /opt/semaphore/roles
```

2. Обновите docker-compose.yml, добавив volumes:
```yaml
volumes:
  - "/path/to/your/ansible/playbooks:/tmp/playbooks"
  - "/path/to/your/ansible/roles:/tmp/roles"
```

## Особенности реализации

1. **Полная интеграция с Ansible**:
   - Поддержка инвентаря, плейбуков и ролей
   - Автоматическая синхронизация конфигураций

2. **Гибкое развертывание**:
   - Поддержка разных окружений (production/staging)
   - Возможность использования существующих структур Ansible

3. **Безопасность**:
   - Переменные чувствительных данных в отдельных файлах
   - Рекомендации по смене паролей по умолчанию

4. **Масштабируемость**:
   - Простое добавление новых плейбуков
   - Поддержка обновления конфигураций

После развертывания Semaphore будет доступен по адресу `http://<сервер>:3000` с полным доступом к вашим плейбукам и инвентарю Ansible.