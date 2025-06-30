Вот Ansible роль для развертывания Semaphore с PostgreSQL на основе предоставленного docker-compose файла:

```yaml
# roles/semaphore/tasks/main.yml
---
- name: Ensure Docker is installed
  include_tasks: install_docker.yml

- name: Create Docker network
  docker_network:
    name: semaphore_network
    driver: bridge
    state: present

- name: Create Docker volumes
  docker_volume:
    name: "{{ item }}"
    state: present
  loop:
    - semaphore_data
    - semaphore_config
    - semaphore_tmp
    - semaphore_postgres

- name: Deploy PostgreSQL container
  docker_container:
    name: semaphore_db
    image: postgres
    env:
      POSTGRES_USER: semaphore
      POSTGRES_PASSWORD: y4teq5ojw7
      POSTGRES_DB: semaphore
    volumes:
      - semaphore_postgres:/var/lib/postgresql/data
    networks:
      - name: semaphore_network
    restart_policy: unless-stopped
    state: started

- name: Deploy Semaphore container
  docker_container:
    name: semaphore
    image: semaphoreui/semaphore:v2.15.0
    ports:
      - "3000:3000"
    env:
      SEMAPHORE_DB_DIALECT: postgres
      SEMAPHORE_DB_HOST: semaphore_db
      SEMAPHORE_DB_NAME: semaphore
      SEMAPHORE_DB_USER: semaphore
      SEMAPHORE_DB_PASS: y4teq5ojw7
      SEMAPHORE_ADMIN: admin
      SEMAPHORE_ADMIN_PASSWORD: admin123!
      SEMAPHORE_ADMIN_NAME: Admin
      SEMAPHORE_ADMIN_EMAIL: admin@localhost
      SEMAPHORE_USE_REMOTE_RUNNER: "True"
      SEMAPHORE_RUNNER_REGISTRATION_TOKEN: "i+IYthCbXvolssK5Dzf1v6nxGDQamedkoDuJpx8Av3A="
      ANSIBLE_HOST_KEY_CHECKING: "True"
    volumes:
      - semaphore_data:/var/lib/semaphore
      - semaphore_config:/etc/semaphore
      - semaphore_tmp:/tmp/semaphore
    networks:
      - name: semaphore_network
    restart_policy: unless-stopped
    state: started
    depends_on:
      - semaphore_db
```

```yaml
# roles/semaphore/tasks/install_docker.yml
---
- name: Install prerequisites
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - software-properties-common
    state: present
    update_cache: yes
  when: ansible_os_family == 'Debian'

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present
  when: ansible_os_family == 'Debian'

- name: Add Docker repository
  apt_repository:
    repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable
    state: present
  when: ansible_os_family == 'Debian'

- name: Install Docker
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present
    update_cache: yes
  when: ansible_os_family == 'Debian'

- name: Install Docker (RedHat)
  yum:
    name:
      - yum-utils
      - device-mapper-persistent-data
      - lvm2
    state: present
  when: ansible_os_family == 'RedHat'

- name: Add Docker repository (RedHat)
  get_url:
    url: https://download.docker.com/linux/centos/docker-ce.repo
    dest: /etc/yum.repos.d/docker-ce.repo
  when: ansible_os_family == 'RedHat'

- name: Install Docker (RedHat)
  yum:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
    state: present
  when: ansible_os_family == 'RedHat'

- name: Start and enable Docker service
  service:
    name: docker
    state: started
    enabled: yes

- name: Install Docker Python module
  pip:
    name: docker
    state: present
```

```yaml
# roles/semaphore/defaults/main.yml
---
# Semaphore configuration
semaphore_image: semaphoreui/semaphore:v2.15.0
semaphore_port: 3000

# PostgreSQL configuration
postgres_image: postgres
postgres_user: semaphore
postgres_password: y4teq5ojw7
postgres_db: semaphore

# Admin credentials
semaphore_admin: admin
semaphore_admin_password: admin123!
semaphore_admin_name: Admin
semaphore_admin_email: admin@localhost

# Runner configuration
semaphore_use_remote_runner: "True"
semaphore_runner_token: "i+IYthCbXvolssK5Dzf1v6nxGDQamedkoDuJpx8Av3A="
```

```yaml
# roles/semaphore/meta/main.yml
---
dependencies: []
```

Для использования этой роли:

1. Создайте структуру каталогов:
```
roles/
  semaphore/
    tasks/
      main.yml
      install_docker.yml
    defaults/
      main.yml
    meta/
      main.yml
```

2. Создайте playbook для использования роли:

```yaml
# deploy_semaphore.yml
---
- hosts: all
  become: yes
  roles:
    - semaphore
```

3. Запустите playbook:
```
ansible-playbook -i inventory deploy_semaphore.yml
```

Роль включает:
- Установку Docker (для Debian/Ubuntu и RedHat/CentOS)
- Создание Docker сети и томов
- Развертывание контейнера PostgreSQL
- Развертывание контейнера Semaphore с указанными переменными окружения
- Все настройки из оригинального docker-compose файла

Вы можете переопределить любые параметры из defaults/main.yml при вызове роли или в inventory.


Вот шаблон `docker-compose.yml.j2` для Semaphore с PostgreSQL, который можно использовать в Ansible роли:

```jinja2
# roles/semaphore/templates/docker-compose.yml.j2
version: '3.8'

services:
  semaphore_db:
    image: {{ postgres_image | default('postgres:latest') }}
    environment:
      POSTGRES_USER: {{ postgres_user | default('semaphore') }}
      POSTGRES_PASSWORD: {{ postgres_password | default('y4teq5ojw7') }}
      POSTGRES_DB: {{ postgres_db | default('semaphore') }}
    volumes:
      - semaphore_postgres:/var/lib/postgresql/data
    networks:
      - semaphore_network
    {% if postgres_restart_policy is defined %}
    restart: {{ postgres_restart_policy }}
    {% endif %}

  semaphore:
    image: {{ semaphore_image | default('semaphoreui/semaphore:v2.15.0') }}
    ports:
      - "{{ semaphore_port | default('3000') }}:3000"
    depends_on:
      - semaphore_db
    environment:
      SEMAPHORE_DB_DIALECT: postgres
      SEMAPHORE_DB_HOST: semaphore_db
      SEMAPHORE_DB_NAME: {{ postgres_db | default('semaphore') }}
      SEMAPHORE_DB_USER: {{ postgres_user | default('semaphore') }}
      SEMAPHORE_DB_PASS: {{ postgres_password | default('y4teq5ojw7') }}
      SEMAPHORE_ADMIN: {{ semaphore_admin | default('admin') }}
      SEMAPHORE_ADMIN_PASSWORD: {{ semaphore_admin_password | default('admin123!') }}
      SEMAPHORE_ADMIN_NAME: {{ semaphore_admin_name | default('Admin') }}
      SEMAPHORE_ADMIN_EMAIL: {{ semaphore_admin_email | default('admin@localhost') }}
      SEMAPHORE_USE_REMOTE_RUNNER: "{{ semaphore_use_remote_runner | default('True') }}"
      SEMAPHORE_RUNNER_REGISTRATION_TOKEN: "{{ semaphore_runner_token | default('i+IYthCbXvolssK5Dzf1v6nxGDQamedkoDuJpx8Av3A=') }}"
      ANSIBLE_HOST_KEY_CHECKING: "{{ ansible_host_key_checking | default('True') }}"
    volumes:
      - semaphore_data:/var/lib/semaphore
      - semaphore_config:/etc/semaphore
      - semaphore_tmp:/tmp/semaphore
    networks:
      - semaphore_network
    {% if semaphore_restart_policy is defined %}
    restart: {{ semaphore_restart_policy }}
    {% endif %}

volumes:
  semaphore_data:
  semaphore_config:
  semaphore_tmp:
  semaphore_postgres:

networks:
  semaphore_network:
    driver: bridge
```

Соответствующая Ansible роль будет выглядеть так:

```yaml
# roles/semaphore/tasks/main.yml
---
- name: Ensure Docker and Docker Compose are installed
  include_tasks: install_docker.yml

- name: Create directory for Semaphore
  file:
    path: "{{ semaphore_install_dir | default('/opt/semaphore') }}"
    state: directory
    mode: '0755'

- name: Deploy docker-compose template
  template:
    src: docker-compose.yml.j2
    dest: "{{ semaphore_install_dir | default('/opt/semaphore') }}/docker-compose.yml"
    mode: '0644'

- name: Ensure Semaphore is running
  community.docker.docker_compose:
    project_src: "{{ semaphore_install_dir | default('/opt/semaphore') }}"
    state: present
    recreate: always
    pull: yes
```

```yaml
# roles/semaphore/defaults/main.yml
---
# Docker Compose settings
semaphore_install_dir: /opt/semaphore

# PostgreSQL settings
postgres_image: postgres:latest
postgres_user: semaphore
postgres_password: y4teq5ojw7
postgres_db: semaphore
postgres_restart_policy: unless-stopped

# Semaphore settings
semaphore_image: semaphoreui/semaphore:v2.15.0
semaphore_port: 3000
semaphore_restart_policy: unless-stopped

# Semaphore admin settings
semaphore_admin: admin
semaphore_admin_password: admin123!
semaphore_admin_name: Admin
semaphore_admin_email: admin@localhost

# Runner settings
semaphore_use_remote_runner: "True"
semaphore_runner_token: "i+IYthCbXvolssK5Dzf1v6nxGDQamedkoDuJpx8Av3A="
ansible_host_key_checking: "True"
```

Пример использования роли в playbook:

```yaml
# deploy_semaphore.yml
---
- hosts: semaphore_servers
  become: yes
  vars:
    postgres_password: "{{ vault_postgres_password }}"
    semaphore_admin_password: "{{ vault_semaphore_admin_password }}"
    semaphore_runner_token: "{{ vault_semaphore_runner_token }}"
  roles:
    - semaphore
```

Преимущества такого подхода:
1. Все параметры конфигурируются через переменные
2. Возможность использования vault для секретов
3. Гибкость настройки через inventory
4. Простота обновления - достаточно изменить версию в переменных
5. Поддержка кастомных параметров (restart policy, volumes и т.д.)

Для развертывания:
```bash
ansible-playbook -i inventory deploy_semaphore.yml
```