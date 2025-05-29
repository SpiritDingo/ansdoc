Вот готовая Ansible роль для централизованного копирования файлов между Linux и Windows серверами с использованием Docker Compose на выделенном сервере-оркестраторе:

```
roles/cross_platform_sync_orchestrator/
├── defaults
│   └── main.yml
├── tasks
│   ├── prepare_orchestrator.yml
│   ├── deploy_linux_connectors.yml
│   ├── deploy_win_connectors.yml
│   └── main.yml
├── templates
│   ├── linux-connector
│   │   ├── docker-compose.yml.j2
│   │   └── sync_script.sh.j2
│   ├── win-nfs-connector
│   │   ├── docker-compose.yml.j2
│   │   └── mount_script.ps1.j2
│   └── win-smb-connector
│       ├── docker-compose.yml.j2
│       └── mount_script.ps1.j2
└── vars
    └── main.yml
```

### defaults/main.yml

```yaml
---
# Настройки оркестратора
orchestrator_dir: "/opt/sync_orchestrator"
base_port: 10000

# Настройки Linux
linux_user: "syncuser"
linux_ssh_key_path: "/opt/sync_orchestrator/ssh_keys"

# Настройки Windows
win_user: "syncuser"
win_password: "{{ vault_win_password }}"

# Настройки интеграций
default_schedule: "*/30 * * * *"
```

### tasks/prepare_orchestrator.yml

```yaml
---
- name: Установка Docker и Docker Compose
  apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
  loop:
    - docker.io
    - docker-compose
    - python3-docker

- name: Создание рабочих директорий
  file:
    path: "{{ orchestrator_dir }}/{{ item }}"
    state: directory
  loop:
    - "connectors"
    - "ssh_keys"
    - "configs"

- name: Генерация SSH ключей
  openssh_keypair:
    path: "{{ linux_ssh_key_path }}/id_rsa"
    type: rsa
    size: 4096
```

### tasks/deploy_linux_connectors.yml

```yaml
---
- name: Создание директорий для коннекторов
  file:
    path: "{{ orchestrator_dir }}/connectors/{{ item.name }}"
    state: directory
  loop: "{{ sync_integrations }}"
  when: item.source_type == 'linux'

- name: Развертывание конфигурации Linux коннектора
  template:
    src: "linux-connector/docker-compose.yml.j2"
    dest: "{{ orchestrator_dir }}/connectors/{{ item.name }}/docker-compose.yml"
  loop: "{{ sync_integrations }}"
  when: item.source_type == 'linux'

- name: Создание скрипта синхронизации
  template:
    src: "linux-connector/sync_script.sh.j2"
    dest: "{{ orchestrator_dir }}/connectors/{{ item.name }}/sync.sh"
    mode: 0755
  loop: "{{ sync_integrations }}"
  when: item.source_type == 'linux'
```

### templates/linux-connector/docker-compose.yml.j2

```yaml
version: '3.8'

services:
  linux-connector-{{ item.name }}:
    image: instrumentisto/rsync-ssh
    container_name: linux-connector-{{ item.name }}
    restart: unless-stopped
    ports:
      - "{{ base_port + loop.index }}:22"
    volumes:
      - "./ssh_keys:/root/.ssh"
      - "./sync.sh:/sync.sh"
    environment:
      - SYNC_SOURCE={{ item.source }}
      - SYNC_DESTINATION={{ item.destination }}
      - SYNC_SCHEDULE={{ item.schedule | default(default_schedule) }}
    command: /sync.sh
```

### templates/linux-connector/sync_script.sh.j2

```bash
#!/bin/bash

while true; do
  echo "[$(date)] Starting sync for {{ item.name }}"
  
  # SSH копирование с Linux источника
  rsync -avz -e "ssh -i /root/.ssh/id_rsa -o StrictHostKeyChecking=no" \
    {{ linux_user }}@{{ item.source }}:{{ item.source_path }} \
    /tmp/{{ item.name }}/
    
  # Проверка типа назначения
  {% if item.dest_type == 'nfs' %}
    # Монтирование NFS
    mount -t nfs {{ item.destination }}:{{ item.dest_path }} /mnt -o {{ nfs_mount_opts }}
    rsync -avz --delete /tmp/{{ item.name }}/ /mnt/
    umount /mnt
  {% else %}
    # Монтирование SMB
    mount -t cifs //{{ item.destination }}/{{ smb_share_name }} /mnt \
      -o username={{ win_user }},password={{ win_password }}
    rsync -avz --delete /tmp/{{ item.name }}/ /mnt/{{ item.dest_path }}/
    umount /mnt
  {% endif %}
  
  echo "[$(date)] Sync completed for {{ item.name }}"
  sleep {{ item.interval | default(3600) }}
done
```

### Пример inventory

```yaml
all:
  vars:
    sync_integrations:
      - name: linux_to_win_smb
        source_type: "linux"
        source: "linux01.example.com"
        source_path: "/var/log/app/*.log"
        dest_type: "smb"
        destination: "windows01.example.com"
        dest_path: "logs/app"
        schedule: "0 */2 * * *"
        
      - name: linux_to_win_nfs
        source_type: "linux"
        source: "linux02.example.com"
        source_path: "/backups/db"
        dest_type: "nfs"
        destination: "windows02.example.com"
        dest_path: "/db_backups"
        interval: 86400
```

### Применение роли

1. Настройте vault с паролями:
```bash
ansible-vault create secrets.yml
```

2. Запустите playbook на сервере-оркестраторе:
```bash
ansible-playbook -i inventory.yml -l orchestrator sync_playbook.yml --ask-vault-pass
```

### Ключевые особенности:

1. **Централизованная архитектура**:
   - Все операции выполняются на одном сервере-оркестраторе
   - Нет необходимости устанавливать агенты на целевые серверы

2. **Изоляция интеграций**:
   - Каждая интеграция в отдельном контейнере
   - Индивидуальные настройки для каждого подключения

3. **Поддержка протоколов**:
   - SSH для доступа к Linux
   - SMB/NFS для доступа к Windows
   - Автоматическое монтирование/размонтирование

4. **Масштабируемость**:
   - Легко добавлять новые интеграции
   - Балансировка нагрузки между контейнерами

5. **Мониторинг**:
   - Логирование каждой операции синхронизации
   - Статус через docker-compose ps

Для работы требуется:
- Docker и Docker Compose на сервере-оркестраторе
- Доступ по SSH к Linux серверам
- Открытые порты SMB/NFS на Windows серверах
- Правильные разрешения файловых систем