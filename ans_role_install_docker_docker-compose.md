# Ansible роль для создания пользователя Docker с Docker Compose v2 на Oracle Linux 9

Полная роль для настройки пользователя Docker с поддержкой Docker Compose v2 (нативной реализации) на Oracle Linux 9.

## Структура роли

```
roles/docker_user_ol9/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── install_docker.yml
│   ├── configure_user.yml
│   └── setup_compose.yml
├── handlers/
│   └── main.yml
└── vars/
    └── OracleLinux.yml
```

## Содержание файлов

### defaults/main.yml

```yaml
---
# Основные параметры пользователя
docker_user: docker_admin
docker_user_uid: 2001
docker_user_groups: "docker"
docker_user_shell: /bin/bash
docker_user_home: "/home/{{ docker_user }}"

# Параметры Docker
docker_edition: "ce"  # ce для Community Edition
docker_compose_version: "v2"  # Используем нативную реализацию Docker Compose v2
docker_repo_url: "https://download.docker.com/linux/centos/docker-ce.repo"

# Параметры системы
manage_firewall: false
manage_selinux: true
```

### vars/OracleLinux.yml

```yaml
---
# Специфичные параметры для Oracle Linux 9
docker_packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-buildx-plugin
  - docker-compose-plugin

prerequisite_packages:
  - yum-utils
  - device-mapper-persistent-data
  - lvm2
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  ansible.builtin.include_vars: "{{ ansible_distribution }}.yml"

- name: Install Docker and prerequisites
  include_tasks: install_docker.yml
  tags: install

- name: Configure Docker user
  include_tasks: configure_user.yml
  tags: user

- name: Setup Docker Compose v2
  include_tasks: setup_compose.yml
  tags: compose
```

### tasks/install_docker.yml

```yaml
---
- name: Install prerequisite packages
  dnf:
    name: "{{ prerequisite_packages }}"
    state: present
    update_cache: yes

- name: Add Docker CE repository
  get_url:
    url: "{{ docker_repo_url }}"
    dest: /etc/yum.repos.d/docker-ce.repo
    validate_certs: no  # Необходимо для некоторых систем Oracle Linux

- name: Install Docker packages
  dnf:
    name: "{{ docker_packages }}"
    state: present
    enablerepo: "docker-ce-*"

- name: Ensure Docker group exists
  group:
    name: docker
    state: present

- name: Start and enable Docker service
  service:
    name: docker
    state: started
    enabled: yes
  notify:
    - restart docker
```

### tasks/configure_user.yml

```yaml
---
- name: Create Docker user account
  user:
    name: "{{ docker_user }}"
    uid: "{{ docker_user_uid }}"
    group: "{{ docker_user }}"
    groups: "{{ docker_user_groups }}"
    append: yes
    shell: "{{ docker_user_shell }}"
    home: "{{ docker_user_home }}"
    create_home: yes
    system: no

- name: Ensure .ssh directory exists
  file:
    path: "{{ docker_user_home }}/.ssh"
    state: directory
    owner: "{{ docker_user }}"
    group: "{{ docker_user }}"
    mode: '0700'

- name: Set proper permissions on Docker socket
  file:
    path: /var/run/docker.sock
    owner: root
    group: docker
    mode: '0660'

- name: Configure SELinux for Docker if enabled
  seboolean:
    name: container_manage_cgroup
    state: yes
    persistent: yes
  when: 
    - manage_selinux | bool
    - ansible_selinux.status == "enabled"
```

### tasks/setup_compose.yml

```yaml
---
- name: Verify Docker Compose v2 is installed (as plugin)
  command: docker compose version
  register: compose_check
  changed_when: false
  ignore_errors: yes

- name: Create symlink for docker-compose (backward compatibility)
  file:
    src: /usr/libexec/docker/cli-plugins/docker-compose
    dest: /usr/local/bin/docker-compose
    state: link
    owner: root
    group: root
  when: compose_check.rc == 0

- name: Create docker-compose projects directory
  file:
    path: "{{ docker_user_home }}/compose-projects"
    state: directory
    owner: "{{ docker_user }}"
    group: "{{ docker_user }}"
    mode: '0755'

- name: Create sample docker-compose.yaml
  copy:
    dest: "{{ docker_user_home }}/compose-projects/sample-compose.yml"
    content: |
      version: '3.8'
      services:
        web:
          image: nginx:alpine
          ports:
            - "8080:80"
        db:
          image: postgres:13
          environment:
            POSTGRES_PASSWORD: example
    owner: "{{ docker_user }}"
    group: "{{ docker_user }}"
    mode: '0644'
```

### handlers/main.yml

```yaml
---
- name: restart docker
  service:
    name: docker
    state: restarted
```

## Пример playbook

```yaml
---
- name: Configure Docker user on Oracle Linux 9
  hosts: ol9_servers
  become: yes
  vars:
    # Переопределение переменных по умолчанию
    docker_user: "ol9_docker"
    docker_user_groups: "docker,wheel"
    manage_firewall: true
  
  pre_tasks:
    - name: Check if Oracle Linux 9
      ansible.builtin.fail:
        msg: "This playbook requires Oracle Linux 9"
      when: ansible_distribution != "OracleLinux" or ansible_distribution_major_version != "9"

  roles:
    - role: docker_user_ol9
      tags: [docker, compose]

  post_tasks:
    - name: Verify Docker installation
      command: docker --version
      register: docker_version
      changed_when: false

    - name: Verify Docker Compose v2
      command: docker compose version
      register: compose_version
      changed_when: false

    - name: Show versions
      debug:
        msg:
          - "Docker: {{ docker_version.stdout }}"
          - "Docker Compose: {{ compose_version.stdout }}"
```

## Особенности реализации

1. **Использование нативного Docker Compose v2**:
   - Встроенная реализация через `docker compose` (не требует отдельной установки)
   - Создается симлинк для обратной совместимости с `docker-compose`

2. **Оптимизация для Oracle Linux 9**:
   - Использование CentOS репозиториев для Docker CE
   - Учет особенностей SELinux
   - Поддержка системных ограничений Oracle Linux

3. **Безопасность**:
   - Правильные права на Docker socket (0660)
   - Отдельный пользователь для работы с Docker
   - Настройка домашнего каталога

4. **Удобство использования**:
   - Создается пример docker-compose.yml
   - Отдельная директория для проектов
   - Проверка версий после установки

Для применения роли:
1. Создайте структуру каталогов
2. Заполните файлы как указано выше
3. Запустите playbook: `ansible-playbook -i inventory setup_docker.yml`