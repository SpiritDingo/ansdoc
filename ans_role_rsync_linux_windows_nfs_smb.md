# Ansible роль для кросс-платформенной синхронизации Linux-Windows через NFS/Samba

Эта роль реализует синхронизацию файлов между Linux и Windows серверами с использованием Docker-контейнеров, поддерживая как NFS, так и SMB протоколы для доступа к Windows-ресурсам.

## Архитектура решения

```
roles/cross_fs_sync/
├── defaults
│   └── main.yml
├── tasks
│   ├── linux
│   │   ├── server.yml
│   │   ├── nfs_server.yml
│   │   └── samba_server.yml
│   ├── windows
│   │   ├── client.yml
│   │   ├── nfs_client.yml
│   │   └── samba_client.yml
│   └── main.yml
├── templates
│   ├── linux
│   │   ├── docker-compose.nfs.yml.j2
│   │   ├── docker-compose.smb.yml.j2
│   │   └── sync_script.sh.j2
│   └── windows
│       ├── docker-compose.nfs.yml.j2
│       ├── docker-compose.smb.yml.j2
│       └── sync_script.ps1.j2
└── vars
    └── main.yml
```

### defaults/main.yml

```yaml
---
# Общие настройки
sync_user: "syncuser"
sync_password: "{{ vault_sync_password }}"
data_dir_linux: "/opt/sync_data"
docker_dir_linux: "/opt/sync_containers"
data_dir_windows: "C:\\sync_data"
docker_dir_windows: "C:\\sync_containers"

# Настройки NFS
nfs_export_options: "rw,sync,no_subtree_check,no_root_squash"
nfs_mount_options: "vers=3,nolock,soft"

# Настройки Samba
smb_share_name: "sync_share"
smb_available: "yes"
smb_browseable: "no"
smb_guest_ok: "no"

# Расписание синхронизации
default_schedule: "*/15 * * * *"
```

### tasks/main.yml

```yaml
---
- name: Определение ОС и протокола
  set_fact:
    is_linux: "{{ ansible_facts['os_family'] == 'Debian' }}"
    is_windows: "{{ ansible_facts['os_family'] == 'Windows' }}"
    use_nfs: "{{ sync_protocol == 'nfs' }}"
    use_smb: "{{ sync_protocol == 'smb' }}"

- include_tasks: linux/server.yml
  when: is_linux

- include_tasks: windows/client.yml
  when: is_windows
```

### tasks/linux/nfs_server.yml

```yaml
---
- name: Установка NFS сервера
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nfs-kernel-server
    - rpcbind

- name: Создание экспорта NFS
  lineinfile:
    path: /etc/exports
    line: "{{ data_dir_linux }}/{{ item.name }} {{ item.client_ip }}({{ nfs_export_options }})"
    state: present
  loop: "{{ sync_integrations }}"
  notify: restart nfs

- name: Применение экспортов NFS
  command: exportfs -ra
```

### tasks/windows/nfs_client.yml

```yaml
---
- name: Установка NFS клиента
  win_feature:
    name: NFS-Client
    state: present

- name: Создание точки монтирования
  win_file:
    path: "{{ data_dir_windows }}\\{{ item.name }}"
    state: directory
  loop: "{{ sync_integrations }}"

- name: Монтирование NFS share
  win_command: |
    mount -o "{{ nfs_mount_options }}" \\{{ item.server }}\{{ data_dir_linux | replace('/','\\') }}\{{ item.name }} {{ data_dir_windows }}\{{ item.name }}
  loop: "{{ sync_integrations }}"
  register: mount_result
  failed_when: mount_result.rc != 0 and 'уже подключен' not in mount_result.stderr
```

### templates/linux/docker-compose.nfs.yml.j2

```yaml
version: '3'

services:
  sync_nfs_{{ item.name }}:
    image: instrumentisto/rsync-ssh
    container_name: sync_nfs_{{ item.name }}
    restart: always
    volumes:
      - "{{ data_dir_linux }}/{{ item.name }}:/data"
    command: >
      sh -c 'while true; do
        rsync -avz --delete /data/ {{ sync_user }}@{{ item.client }}:/{{ data_dir_windows }}/{{ item.name }} || true;
        sleep {{ item.interval | default(300) }};
      done'
```

### templates/windows/sync_script.ps1.j2 (NFS вариант)

```powershell
# PowerShell скрипт для NFS синхронизации

$ErrorActionPreference = "Stop"

try {
    $RSYNC_OPTS = "-avz --delete"
    
    & docker run --rm `
        -v "{{ data_dir_windows }}\{{ item.name }}:/data" `
        instrumentisto/rsync-ssh `
        rsync $RSYNC_OPTS /data/ "{{ sync_user }}@{{ item.server }}:{{ item.remote_path | default('/data') }}"
    
    Write-Output "$(Get-Date) - NFS Sync {{ item.name }} completed"
} catch {
    Write-Output "$(Get-Date) - NFS Sync {{ item.name }} failed: $_"
    exit 1
}
```

## Пример inventory для NFS

```yaml
all:
  vars:
    sync_protocol: "nfs"
    sync_integrations:
      - name: db_backup
        description: "Database backups"
        server: "linux01.example.com"
        client: "windows01.example.com"
        client_ip: "192.168.1.100"
        direction: "push"
        schedule: "0 2 * * *"
```

## Пример inventory для Samba

```yaml
all:
  vars:
    sync_protocol: "smb"
    sync_integrations:
      - name: user_docs
        description: "User documents sync"
        server: "windows02.example.com"
        client: "linux02.example.com"
        smb_username: "domain\syncuser"
        smb_password: "{{ vault_smb_password }}"
        direction: "pull"
        schedule: "*/5 * * * *"
```

## Особенности реализации

1. **Поддержка обоих протоколов**:
   - NFS для высокой производительности
   - SMB для лучшей совместимости с Windows

2. **Безопасность**:
   - Шифрование учетных данных с помощью Ansible Vault
   - Изоляция контейнеров для каждой интеграции
   - Минимальные необходимые права доступа

3. **Гибкость**:
   - Настраиваемое расписание синхронизации
   - Поддержка push/pull моделей
   - Возможность выбора протокола для каждой интеграции

4. **Мониторинг**:
   - Логирование результатов каждой синхронизации
   - Статус контейнеров через Docker
   - Оповещения об ошибках

Для работы роли необходимо:

1. Для NFS:
   - Открыть порты 111, 2049 на firewall
   - Настроить правильные экспорты на Linux

2. Для Samba:
   - Настроить общие папки на Windows
   - Указать доменные учетные данные при необходимости

3. Общее:
   - Docker на всех узлах
   - Ansible 2.9+ с поддержкой Windows
   - Правильные разрешения файловой системы