Вот пример Ansible роли для настройки Docker и Docker Compose в rootless-режиме:

Структура роли:

```
docker-rootless/
├── tasks
│   ├── main.yml
│   ├── prerequisites.yml
│   ├── install.yml
│   ├── configure.yml
│   └── services.yml
├── handlers
│   └── main.yml
├── templates
│   ├── docker-rootless.service.j2
│   └── docker-rootless.socket.j2
└── defaults
    └── main.yml
```

defaults/main.yml:

```yaml
docker_rootless_user: "{{ ansible_user }}"
docker_version: "latest"
docker_compose_version: "v2.27.0"
docker_rootless_install_dir: "/opt/docker-rootless"
docker_bin_dir: "{{ docker_rootless_install_dir }}/bin"
docker_data_dir: "/home/{{ docker_rootless_user }}/.docker"
enable_docker_service: true
subuid_start: 100000
subuid_count: 65536
subgid_start: 100000
subgid_count: 65536
```

tasks/prerequisites.yml:

```yaml
---
- name: "Install rootless prerequisites"
  package:
    name:
      - uidmap
      - dbus-user-session
      - fuse-overlayfs
      - slirp4netns
    state: present

- name: "Configure /etc/subuid"
  lineinfile:
    path: /etc/subuid
    line: "{{ docker_rootless_user }}:{{ subuid_start }}:{{ subuid_count }}"
    state: present
    create: yes

- name: "Configure /etc/subgid"
  lineinfile:
    path: /etc/subgid
    line: "{{ docker_rootless_user }}:{{ subgid_start }}:{{ subgid_count }}"
    state: present
    create: yes

- name: "Ensure user login session"
  command: loginctl enable-linger {{ docker_rootless_user }}
  changed_when: false
```

tasks/install.yml:

```yaml
---
- name: "Create installation directories"
  file:
    path: "{{ docker_rootless_install_dir }}"
    state: directory
    mode: '0755'

- name: "Download Docker rootless install script"
  get_url:
    url: "https://get.docker.com/rootless"
    dest: "{{ docker_rootless_install_dir }}/install.sh"
    mode: '0755'

- name: "Install Docker rootless"
  become_user: "{{ docker_rootless_user }}"
  environment:
    PATH: "{{ docker_bin_dir }}:{{ ansible_env.PATH }}"
  command: "{{ docker_rootless_install_dir }}/install.sh"
  args:
    creates: "{{ docker_bin_dir }}/docker"

- name: "Download Docker Compose"
  get_url:
    url: "https://github.com/docker/compose/releases/download/{{ docker_compose_version }}/docker-compose-linux-x86_64"
    dest: "{{ docker_bin_dir }}/docker-compose"
    mode: '0755'

- name: "Create symlinks for user"
  file:
    src: "{{ docker_bin_dir }}/{{ item }}"
    dest: "/home/{{ docker_rootless_user }}/.local/bin/{{ item }}"
    state: link
    force: yes
  loop:
    - docker
    - docker-compose
```

tasks/configure.yml:

```yaml
---
- name: "Set environment variables"
  template:
    src: docker-env.sh.j2
    dest: "/home/{{ docker_rootless_user }}/.docker/env.sh"
    mode: '0644'

- name: "Update shell profiles"
  blockinfile:
    path: "/home/{{ docker_rootless_user }}/.bashrc"
    block: |
      # Docker rootless
      export PATH="{{ docker_bin_dir }}:$PATH"
      [ -f "/home/{{ docker_rootless_user }}/.docker/env.sh" ] && source "/home/{{ docker_rootless_user }}/.docker/env.sh"
    marker: "# {mark} ANSIBLE MANAGED DOCKER ROOTLESS"

- name: "Configure Docker daemon"
  template:
    src: daemon.json.j2
    dest: "{{ docker_data_dir }}/daemon.json"
    mode: '0644'
  become_user: "{{ docker_rootless_user }}"
```

templates/docker-env.sh.j2:

```bash
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock
export PATH="{{ docker_bin_dir }}:$PATH"
export DOCKER_CONFIG="{{ docker_data_dir }}"
```

templates/daemon.json.j2:

```json
{
  "data-root": "{{ docker_data_dir }}/docker",
  "userns-remap": "default",
  "log-driver": "json-file",
  "storage-driver": "overlay2",
  "userland-proxy": false,
  "iptables": true,
  "ip-forward": true
}
```

tasks/services.yml:

```yaml
---
- name: "Install systemd service"
  template:
    src: docker-rootless.service.j2
    dest: /etc/systemd/system/docker-rootless.service
    mode: '0644'
  when: enable_docker_service

- name: "Install systemd socket"
  template:
    src: docker-rootless.socket.j2
    dest: /etc/systemd/system/docker-rootless.socket
    mode: '0644'
  when: enable_docker_service

- name: "Reload systemd"
  systemd:
    daemon_reload: yes
  when: enable_docker_service

- name: "Enable and start Docker rootless service"
  systemd:
    name: docker-rootless
    state: started
    enabled: yes
    daemon_reload: yes
  when: enable_docker_service
```

templates/docker-rootless.service.j2:

```ini
[Unit]
Description=Docker Rootless
After=network.target docker-rootless.socket

[Service]
Type=notify
User={{ docker_rootless_user }}
Group={{ docker_rootless_user }}
Environment=PATH={{ docker_bin_dir }}:/usr/bin:/bin
Environment=DOCKER_HOST=unix:///run/user/%U/docker.sock
ExecStart={{ docker_bin_dir }}/dockerd-rootless.sh
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

[Install]
WantedBy=default.target
```

templates/docker-rootless.socket.j2:

```ini
[Unit]
Description=Docker Rootless Socket

[Socket]
ListenStream=/run/user/%U/docker.sock
SocketMode=0660

[Install]
WantedBy=sockets.target
```

tasks/main.yml:

```yaml
---
- include_tasks: prerequisites.yml
  tags: prerequisites

- include_tasks: install.yml
  tags: install

- include_tasks: configure.yml
  tags: configure

- include_tasks: services.yml
  tags: services
```

handlers/main.yml:

```yaml
---
- name: restart docker-rootless
  systemd:
    name: docker-rootless
    state: restarted
    daemon_reload: yes

- name: reload systemd
  systemd:
    daemon_reload: yes
```

Пример использования в плейбуке:

```yaml
- hosts: servers
  become: yes
  roles:
    - role: docker-rootless
      vars:
        docker_rootless_user: "devuser"
        docker_compose_version: "v2.27.0"
        enable_docker_service: true
```

Основные особенности роли:

1. Безопасность: Работает от непривилегированного пользователя
2. Изоляция: Использует user namespace для изоляции контейнеров
3. Автоматизация: Настраивает все необходимые компоненты
4. Systemd integration: Создает службы для автоматического запуска
5. Гибкость: Позволяет настраивать параметры через переменные

Примечания:

· Требует включенного user namespace в ядре
· Для работы нужны дополнительные пакеты (slirp4netns, fuse-overlayfs)
· Порт для внешнего доступа по умолчанию 2376 (rootless mode)
· Данные хранятся в домашней директории пользователя

Роль можно расширить добавлением:

· Настройки firewall правил
· Мониторинга
· Резервного копирования
· SSL сертификатов для удаленного доступа