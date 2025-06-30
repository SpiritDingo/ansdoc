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