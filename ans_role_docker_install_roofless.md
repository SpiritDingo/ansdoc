Вот Ansible роль для установки Docker и Docker Compose с rootless режимом для Oracle Linux 9 и Ubuntu 22.04/24.04:

Структура роли

```
docker-rootless/
├── tasks/
│   ├── main.yml
│   ├── install_docker.yml
│   ├── install_docker_compose.yml
│   └── setup_rootless.yml
├── defaults/
│   └── main.yml
└── vars/
    ├── OracleLinux9.yml
    └── Ubuntu.yml
```

defaults/main.yml

```yaml
---
# Версии ПО
docker_compose_version: "v2.24.5"
docker_rootless_version: "v26.1.3"

# Пользователь для rootless Docker
docker_rootless_user: "{{ ansible_user | default('vagrant') }}"
docker_rootless_user_group: "{{ docker_rootless_user }}"

# Настройки
docker_data_dir: "/var/lib/docker"
docker_rootless_data_dir: "/home/{{ docker_rootless_user }}/.local/share/docker"
disable_docker_system_service: true
install_docker_compose_standalone: true
```

vars/OracleLinux9.yml

```yaml
---
docker_packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-buildx-plugin
  - docker-compose-plugin

docker_rootless_dependencies:
  - fuse-overlayfs
  - slirp4netns
  - iptables
  - shadow-utils
  - shadow-utils-subid
  - crun
  - conmon
  - dbus-user-session
```

vars/Ubuntu.yml

```yaml
---
docker_packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-buildx-plugin
  - docker-compose-plugin

docker_rootless_dependencies:
  - fuse-overlayfs
  - slirp4netns
  - iptables
  - uidmap
  - dbus-user-session
  - criu
  - podman
```

tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_distribution }}{{ ansible_distribution_major_version }}.yml"

- name: Include installation tasks
  include_tasks: install_docker.yml

- name: Include Docker Compose installation
  include_tasks: install_docker_compose.yml
  when: install_docker_compose_standalone

- name: Include rootless setup
  include_tasks: setup_rootless.yml
```

tasks/install_docker.yml

```yaml
---
- name: Install dependencies for Docker
  package:
    name: "{{ docker_rootless_dependencies }}"
    state: present
  when: docker_rootless_dependencies is defined

- name: Install Docker packages
  package:
    name: "{{ docker_packages }}"
    state: present

- name: Ensure Docker user group exists
  group:
    name: docker
    state: present

- name: Add user to docker group (for fallback)
  user:
    name: "{{ docker_rootless_user }}"
    groups: docker
    append: yes

- name: Disable Docker system service if required
  systemd:
    name: docker
    state: stopped
    enabled: false
    daemon_reload: yes
  when: disable_docker_system_service

- name: Ensure Docker directories exist
  file:
    path: "{{ item }}"
    state: directory
    owner: root
    group: root
    mode: '0755'
  loop:
    - /etc/docker
    - "{{ docker_data_dir }}"

- name: Configure Docker daemon (optional, for system Docker)
  copy:
    dest: /etc/docker/daemon.json
    content: |
      {
        "data-root": "{{ docker_data_dir }}",
        "log-driver": "json-file",
        "log-opts": {
          "max-size": "10m",
          "max-file": "3"
        }
      }
    mode: '0644'
  when: not disable_docker_system_service
```

tasks/install_docker_compose.yml

```yaml
---
- name: Download Docker Compose binary
  get_url:
    url: "https://github.com/docker/compose/releases/download/{{ docker_compose_version }}/docker-compose-{{ ansible_system | lower }}-{{ ansible_machine }}"
    dest: /usr/local/bin/docker-compose
    mode: '0755'
    owner: root
    group: root
  register: download_compose

- name: Create symlink for Docker Compose (for rootless compatibility)
  file:
    src: /usr/local/bin/docker-compose
    dest: "{{ install_docker_compose_standalone | ternary('/usr/bin/docker-compose', '/usr/bin/docker-compose') }}"
    state: link
    force: yes
  when: download_compose.changed
```

tasks/setup_rootless.yml

```yaml
---
- name: Check if user exists
  user:
    name: "{{ docker_rootless_user }}"
    state: present
    create_home: yes
    shell: /bin/bash
  register: user_created

- name: Configure subuid and subgid for rootless Docker
  block:
    - name: Set subuid for user
      lineinfile:
        path: /etc/subuid
        line: "{{ docker_rootless_user }}:100000:65536"
        state: present
        create: yes

    - name: Set subgid for user
      lineinfile:
        path: /etc/subgid
        line: "{{ docker_rootless_user }}:100000:65536"
        state: present
        create: yes
  become: true

- name: Install Docker rootless extras package
  package:
    name: docker-ce-rootless-extras
    state: present

- name: Install rootless Docker as user
  become_user: "{{ docker_rootless_user }}"
  environment:
    XDG_RUNTIME_DIR: "/run/user/{{ ansible_user_id }}"
  block:
    - name: Download and install Docker rootless
      get_url:
        url: "https://download.docker.com/linux/static/stable/{{ ansible_machine }}/docker-{{ docker_rootless_version }}.tgz"
        dest: "/home/{{ docker_rootless_user }}/docker-rootless.tar.gz"
        mode: '0644'

    - name: Extract Docker rootless
      unarchive:
        src: "/home/{{ docker_rootless_user }}/docker-rootless.tar.gz"
        dest: "/home/{{ docker_rootless_user }}/.local/bin"
        remote_src: yes
        extra_opts: ["--strip-components=1"]

    - name: Clean up downloaded archive
      file:
        path: "/home/{{ docker_rootless_user }}/docker-rootless.tar.gz"
        state: absent

- name: Install Docker rootless systemd service
  become_user: "{{ docker_rootless_user }}"
  environment:
    XDG_RUNTIME_DIR: "/run/user/{{ ansible_user_id }}"
    PATH: "/home/{{ docker_rootless_user }}/.local/bin:{{ ansible_env.PATH }}"
  block:
    - name: Install rootless systemd service
      command: "dockerd-rootless-setuptool.sh install"
      args:
        creates: "/home/{{ docker_rootless_user }}/.config/systemd/user/docker.service"

    - name: Enable lingering for user
      command: "loginctl enable-linger {{ docker_rootless_user }}"

- name: Set environment variables for user
  blockinfile:
    path: "/home/{{ docker_rootless_user }}/.bashrc"
    block: |
      # Docker rootless settings
      export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
      export PATH="/home/{{ docker_rootless_user }}/.local/bin:$PATH"
    marker: "# {mark} ANSIBLE MANAGED BLOCK - DOCKER ROOTLESS"

- name: Create Docker CLI configuration for user
  become_user: "{{ docker_rootless_user }}"
  block:
    - name: Ensure .docker directory exists
      file:
        path: "/home/{{ docker_rootless_user }}/.docker"
        state: directory
        mode: '0755'

    - name: Create Docker CLI config
      copy:
        dest: "/home/{{ docker_rootless_user }}/.docker/config.json"
        content: |
          {
            "auths": {},
            "credsStore": "pass"
          }
        mode: '0600'

- name: Configure Docker rootless storage driver
  become_user: "{{ docker_rootless_user }}"
  block:
    - name: Ensure Docker rootless config directory exists
      file:
        path: "/home/{{ docker_rootless_user }}/.config/docker"
        state: directory
        mode: '0755'

    - name: Create rootless Docker daemon config
      copy:
        dest: "/home/{{ docker_rootless_user }}/.config/docker/daemon.json"
        content: |
          {
            "data-root": "{{ docker_rootless_data_dir }}",
            "log-driver": "json-file",
            "log-opts": {
              "max-size": "10m",
              "max-file": "3"
            },
            "storage-driver": "overlay2",
            "userland-proxy": false,
            "iptables": true
          }
        mode: '0644'

- name: Ensure proper permissions for user directories
  file:
    path: "/home/{{ docker_rootless_user }}"
    owner: "{{ docker_rootless_user }}"
    group: "{{ docker_rootless_user_group }}"
    recurse: yes

- name: Start Docker rootless service
  become_user: "{{ docker_rootless_user }}"
  environment:
    XDG_RUNTIME_DIR: "/run/user/{{ ansible_user_id }}"
  systemd:
    name: docker
    scope: user
    state: started
    enabled: yes
    daemon_reload: yes

- name: Test Docker rootless installation
  become_user: "{{ docker_rootless_user }}"
  environment:
    DOCKER_HOST: unix:///run/user/{{ ansible_user_id }}/docker.sock
    PATH: "/home/{{ docker_rootless_user }}/.local/bin:{{ ansible_env.PATH }}"
  command: "docker --version"
  register: docker_version
  changed_when: false

- name: Display Docker version
  debug:
    msg: "Docker rootless installed: {{ docker_version.stdout }}"

- name: Test Docker Compose installation
  become_user: "{{ docker_rootless_user }}"
  environment:
    DOCKER_HOST: unix:///run/user/{{ ansible_user_id }}/docker.sock
    PATH: "/home/{{ docker_rootless_user }}/.local/bin:{{ ansible_env.PATH }}"
  command: "docker-compose --version"
  register: compose_version
  changed_when: false

- name: Display Docker Compose version
  debug:
    msg: "Docker Compose installed: {{ compose_version.stdout }}"
```

Пример playbook для использования роли

```yaml
---
- name: Install Docker and Docker Compose with rootless mode
  hosts: all
  become: true
  vars:
    docker_rootless_user: "appuser"  # Пользователь для rootless Docker
    docker_compose_version: "v2.24.5"
    
  roles:
    - docker-rootless

  tasks:
    - name: Verify Docker rootless is working
      become_user: "{{ docker_rootless_user }}"
      environment:
        DOCKER_HOST: unix:///run/user/$(id -u)/docker.sock
      command: "docker run hello-world"
      register: hello_world
      changed_when: false

    - name: Display verification result
      debug:
        msg: "{{ hello_world.stdout }}"
```

Особенности:

1. Поддержка нескольких ОС: Oracle Linux 9, Ubuntu 22.04 и 24.04
2. Rootless Docker: Docker работает от имени обычного пользователя без привилегий root
3. Автоматическая настройка: Настраиваются subuid/subgid, systemd service, переменные окружения
4. Отключение системного Docker: По умолчанию системный Docker служба отключается
5. Docker Compose: Устанавливается отдельный бинарник для совместимости

Использование:

```bash
# Создайте inventory файл
cat > hosts.ini << EOF
[servers]
server1 ansible_host=192.168.1.100 ansible_user=root
server2 ansible_host=192.168.1.101 ansible_user=ubuntu
EOF

# Запустите playbook
ansible-playbook -i hosts.ini playbook.yml
```

После установки пользователь может использовать Docker без sudo:

```bash
# Как пользователь, указанный в docker_rootless_user
docker ps
docker-compose up
```

Роль автоматически настраивает все необходимые компоненты для безопасной работы Docker в rootless-режиме.