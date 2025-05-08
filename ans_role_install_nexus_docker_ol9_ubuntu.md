Вот Ansible роли для выполнения ваших задач на Oracle Linux 9 и Ubuntu. Я разделил их на три части, как вы указали.

### Структура проекта
```
ansible-nexus-docker/
├── inventories/
│   └── production/
│       ├── group_vars/
│       │   └── all.yml
│       └── hosts
├── roles/
│   ├── docker_user/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── vars/
│   │       └── main.yml
│   ├── docker_install/
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   └── vars/
│   │       ├── Deb.yml
│   │       └── RedHat.yml
│   └── nexus_setup/
│       ├── files/
│       │   └── docker-compose.yml
│       ├── tasks/
│       │   └── main.yml
│       └── vars/
│           └── main.yml
└── site.yml
```

### 1. Роль для создания пользователя docker (roles/docker_user)

`roles/docker_user/tasks/main.yml`:
```yaml
---
- name: Ensure docker group exists
  group:
    name: "{{ docker_group }}"
    state: present

- name: Create docker user
  user:
    name: "{{ docker_user }}"
    group: "{{ docker_group }}"
    shell: /bin/bash
    append: yes
    create_home: yes
    home: "/home/{{ docker_user }}"

- name: Add docker user to sudoers
  copy:
    dest: "/etc/sudoers.d/{{ docker_user }}"
    content: "{{ docker_user }} ALL=(ALL) NOPASSWD:ALL"
    validate: "visudo -cf %s"

- name: Ensure .ssh directory exists
  file:
    path: "/home/{{ docker_user }}/.ssh"
    state: directory
    owner: "{{ docker_user }}"
    group: "{{ docker_group }}"
    mode: '0700'
```

`roles/docker_user/vars/main.yml`:
```yaml
docker_user: docker
docker_group: docker
```

### 2. Роль для установки Docker и Docker Compose (roles/docker_install)

`roles/docker_install/tasks/main.yml`:
```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: Install prerequisites
  package:
    name: "{{ docker_prerequisites }}"
    state: present

- name: Add Docker GPG key
  apt_key:
    url: "{{ docker_gpg_key }}"
    state: present
  when: ansible_os_family == 'Debian'

- name: Add Docker repository (Debian)
  apt_repository:
    repo: "{{ docker_repo }}"
    state: present
    update_cache: yes
  when: ansible_os_family == 'Debian'

- name: Add Docker repository (RedHat)
  yum_repository:
    name: docker-ce
    description: Docker CE
    baseurl: "{{ docker_repo }}"
    gpgkey: "{{ docker_gpg_key }}"
    gpgcheck: yes
    enabled: yes
  when: ansible_os_family == 'RedHat'

- name: Install Docker packages
  package:
    name: "{{ docker_packages }}"
    state: present
    update_cache: yes

- name: Ensure Docker is running and enabled
  service:
    name: docker
    state: started
    enabled: yes

- name: Install Docker Compose
  get_url:
    url: "{{ docker_compose_url }}"
    dest: /usr/local/bin/docker-compose
    mode: '0755'

- name: Add docker user to docker group
  user:
    name: "{{ docker_user }}"
    groups: docker
    append: yes

- name: Validate Docker installation
  command: docker --version
  register: docker_version
  changed_when: false

- name: Validate Docker Compose installation
  command: docker-compose --version
  register: docker_compose_version
  changed_when: false

- name: Show Docker versions
  debug:
    msg: "Docker {{ docker_version.stdout }}, Docker Compose {{ docker_compose_version.stdout }}"
```

`roles/docker_install/vars/Deb.yml` (для Ubuntu):
```yaml
docker_prerequisites:
  - apt-transport-https
  - ca-certificates
  - curl
  - gnupg
  - lsb-release
  - software-properties-common

docker_gpg_key: https://download.docker.com/linux/ubuntu/gpg
docker_repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
docker_packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io

docker_compose_url: "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-linux-x86_64"
```

`roles/docker_install/vars/RedHat.yml` (для Oracle Linux 9):
```yaml
docker_prerequisites:
  - yum-utils
  - device-mapper-persistent-data
  - lvm2

docker_gpg_key: https://download.docker.com/linux/centos/gpg
docker_repo: https://download.docker.com/linux/centos/docker-ce.repo
docker_packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io

docker_compose_url: "https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-linux-x86_64"
```

### 3. Роль для установки Nexus CE (roles/nexus_setup)

`roles/nexus_setup/tasks/main.yml`:
```yaml
---
- name: Create directories for Nexus
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ docker_user }}"
    group: "{{ docker_group }}"
    mode: '0755'
  loop:
    - "{{ nexus_data_dir }}"
    - "{{ nexus_data_dir }}/nexus-data"
    - "{{ nexus_data_dir }}/sonatype-work"

- name: Deploy docker-compose.yml for Nexus
  template:
    src: docker-compose.yml
    dest: "{{ nexus_data_dir }}/docker-compose.yml"
    owner: "{{ docker_user }}"
    group: "{{ docker_group }}"
    mode: '0644'

- name: Change ownership of Nexus directory
  file:
    path: "{{ nexus_data_dir }}"
    state: directory
    recurse: yes
    owner: "{{ docker_user }}"
    group: "{{ docker_group }}"

- name: Start Nexus container
  become: yes
  become_user: "{{ docker_user }}"
  community.docker.docker_compose:
    project_src: "{{ nexus_data_dir }}"
    state: present
    pull: yes
    recreate: always

- name: Wait for Nexus to start
  uri:
    url: "http://localhost:{{ nexus_port }}/service/rest/v1/status"
    method: GET
    status_code: 200
    timeout: 30
  register: result
  until: result.status == 200
  retries: 10
  delay: 30
  ignore_errors: yes

- name: Display Nexus info
  debug:
    msg: "Nexus is available at http://{{ ansible_host }}:{{ nexus_port }} - Default admin password is in {{ nexus_data_dir }}/nexus-data/admin.password"
```

`roles/nexus_setup/files/docker-compose.yml`:
```yaml
version: '3'

services:
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    restart: unless-stopped
    ports:
      - "{{ nexus_port }}:8081"
    volumes:
      - "{{ nexus_data_dir }}/nexus-data:/nexus-data"
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2703m -Xmx2703m -XX:MaxDirectMemorySize=2703m
```

`roles/nexus_setup/vars/main.yml`:
```yaml
nexus_port: 8081
nexus_data_dir: /opt/nexus
```

### Основной playbook (site.yml)

```yaml
---
- name: Configure Docker and Nexus on all servers
  hosts: all
  become: yes
  roles:
    - docker_user
    - docker_install
    - nexus_setup
```

### Инвентарь (inventories/production/hosts)

```ini
[servers]
ol9-server ansible_host=192.168.1.100 ansible_os_family=RedHat
ubuntu-server ansible_host=192.168.1.101 ansible_os_family=Debian

[servers:vars]
ansible_user=admin
ansible_ssh_private_key_file=~/.ssh/id_rsa
```

### Глобальные переменные (inventories/production/group_vars/all.yml)

```yaml
---
# Общие переменные для всех хостов
docker_user: docker
docker_group: docker
```

### Использование

1. Установите необходимые коллекции:
```bash
ansible-galaxy collection install community.docker
```

2. Запустите playbook:
```bash
ansible-playbook -i inventories/production/hosts site.yml
```

Этот набор ролей:
1. Создаст пользователя `docker` с необходимыми правами
2. Установит Docker и Docker Compose (разными способами для Ubuntu и Oracle Linux)
3. Развернет Nexus CE в Docker Compose с правильными настройками

Вы можете настроить параметры Nexus (порт, путь к данным) в `roles/nexus_setup/vars/main.yml`.