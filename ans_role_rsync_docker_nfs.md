# Ansible роль для копирования файлов между Linux и Windows через NFS с использованием rsync в Docker

Эта роль предназначена для копирования файлов между Linux-хостами и Windows-машинами с общим доступом NFS, используя Docker-контейнеры с rsync.

## Структура роли

```
roles/rsync_linux_to_win_nfs/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── templates/
│   └── docker-compose.rsync.yml.j2
└── vars/
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки источника (Linux)
linux_src_path: "/path/on/linux"
linux_host: "linux-host"
linux_user: "user"
linux_ssh_port: 22

# Настройки назначения (Windows NFS)
win_nfs_share: "//windows-host/share"
win_nfs_mount_point: "/mnt/windows_share"
win_nfs_options: "vers=3,nolock,noacl"

# Настройки rsync
rsync_options: "-avz --progress --delete"
rsync_exclude: ".git/ tmp/"

# Настройки Docker
rsync_container_image: "alpine:latest"
rsync_container_name: "ansible_rsync_temp"
```

### tasks/main.yml

```yaml
---
- name: Install required packages on Linux host
  become: yes
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nfs-common
    - sshpass
  when: ansible_os_family == 'Debian'

- name: Create Windows NFS mount point
  become: yes
  file:
    path: "{{ win_nfs_mount_point }}"
    state: directory
    mode: '0777'

- name: Mount Windows NFS share
  become: yes
  mount:
    path: "{{ win_nfs_mount_point }}"
    src: "{{ win_nfs_share }}"
    fstype: nfs
    opts: "{{ win_nfs_options }}"
    state: mounted

- name: Generate docker-compose file for rsync
  template:
    src: "docker-compose.rsync.yml.j2"
    dest: "/tmp/docker-compose.rsync.yml"

- name: Start rsync container
  command: "docker-compose -f /tmp/docker-compose.rsync.yml up -d"

- name: Execute rsync from Linux to Windows NFS
  command: >
    docker exec {{ rsync_container_name }} 
    rsync {{ rsync_options }} 
    {% if rsync_exclude %}--exclude='{{ rsync_exclude }}'{% endif %}
    "{{ linux_user }}@{{ linux_host }}:{{ linux_src_path }}/" 
    "{{ win_nfs_mount_point }}/"
  register: rsync_result
  failed_when: rsync_result.rc not in [0, 23, 24]  # 23/24 - partial transfers OK

- name: Unmount Windows NFS share
  become: yes
  mount:
    path: "{{ win_nfs_mount_point }}"
    state: absent

- name: Remove rsync container
  command: "docker-compose -f /tmp/docker-compose.rsync.yml down"
```

### templates/docker-compose.rsync.yml.j2

```yaml
version: '3'

services:
  {{ rsync_container_name }}:
    image: {{ rsync_container_image }}
    container_name: {{ rsync_container_name }}
    volumes:
      - "{{ win_nfs_mount_point }}:{{ win_nfs_mount_point }}"
    environment:
      - SSH_AUTH_SOCK=/tmp/ssh_auth_sock
    command: >
      sh -c "
      apk add --no-cache rsync openssh-client &&
      mkdir -p ~/.ssh &&
      ssh-keyscan -p {{ linux_ssh_port }} {{ linux_host }} >> ~/.ssh/known_hosts &&
      tail -f /dev/null"
```

## Пример использования роли

```yaml
- hosts: linux_gateway
  roles:
    - role: rsync_linux_to_win_nfs
      vars:
        linux_src_path: "/data/project_files"
        linux_host: "dev-linux-server"
        linux_user: "deploy"
        
        win_nfs_share: "//win-fileserver/project_share"
        win_nfs_mount_point: "/mnt/win_share"
        
        rsync_options: "-avz --progress --delete --chmod=0777"
        rsync_exclude: "*.tmp cache/ .git/"
```

## Дополнительные настройки

### Для работы с SSH ключами:

1. Добавьте задачу для копирования SSH ключа в контейнер:

```yaml
- name: Copy SSH key to container
  command: >
    docker cp "{{ ssh_private_key_path }}" 
    {{ rsync_container_name }}:/root/.ssh/id_rsa
  when: use_ssh_key
```

2. Обновите docker-compose шаблон для настройки прав на ключ:

```yaml
command: >
  sh -c "
  apk add --no-cache rsync openssh-client &&
  mkdir -p ~/.ssh &&
  chmod 700 ~/.ssh &&
  ssh-keyscan -p {{ linux_ssh_port }} {{ linux_host }} >> ~/.ssh/known_hosts &&
  {% if use_ssh_key %}
  chmod 600 ~/.ssh/id_rsa &&
  {% endif %}
  tail -f /dev/null"
```

### Для обработки прав файлов в Windows:

Добавьте в rsync_options:
```yaml
rsync_options: "-avz --progress --delete --no-perms --no-owner --no-group"
```

## Особенности реализации

1. Роль монтирует Windows NFS share на Linux-хосте перед копированием
2. Используется временный Docker-контейнер с Alpine Linux для выполнения rsync
3. Контейнер содержит все необходимые утилиты (rsync, ssh)
4. После выполнения NFS share автоматически отмонтируется
5. Поддерживается как аутентификация по паролю, так и по SSH-ключу

Для работы этой роли необходимо:
- Настроить NFS сервер на Windows
- Разрешить подключение к Linux-хосту по SSH
- Установить Docker на хост, где выполняется роль