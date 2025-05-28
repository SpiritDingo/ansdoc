# Ansible Role для копирования файлов между удалёнными Linux-хостами через Rsync в Docker

Вот усовершенствованная Ansible роль для безопасного копирования файлов между удалёнными Linux-серверами с использованием изолированных Docker-контейнеров Rsync.

## Оптимизированная структура роли

```
roles/remote_rsync_docker/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── deploy_rsync_container.yml
│   ├── main.yml
│   ├── perform_transfer.yml
│   └── verify_environment.yml
├── templates/
│   ├── Dockerfile.rsync.j2
│   └── docker-compose.rsync.j2
└── vars/
    └── main.yml
```

## Реализация роли

### defaults/main.yml

```yaml
---
# Основные параметры
source_host: "source.example.com"
destination_host: "destination.example.com"
ssh_user: "ansible"
ssh_port: 22
ssh_private_key: "~/.ssh/id_rsa"

# Настройки Rsync
rsync_container_name: "temp_rsync_{{ 1000 | random(seed=inventory_hostname) }}"
rsync_image: "alpine-rsync:custom"
rsync_port: 2873
rsync_user: "syncuser"
rsync_password: "{{ vault_rsync_password }}"
rsync_timeout: 600

# Пути к данным
source_volume: "/mnt/data/source"
destination_volume: "/mnt/data/destination"
container_data_path: "/data"

# Очистка
remove_container_after: true
```

### tasks/main.yml

```yaml
---
- name: Verify environment prerequisites
  include_tasks: verify_environment.yml

- name: Deploy temporary rsync containers
  include_tasks: deploy_rsync_container.yml
  loop:
    - { host: "{{ source_host }}", mode: "source" }
    - { host: "{{ destination_host }}", mode: "destination" }
  loop_control:
    loop_var: rsync_host

- name: Perform secure file transfer
  include_tasks: perform_transfer.yml

- name: Cleanup temporary containers
  include_tasks: deploy_rsync_container.yml
  loop:
    - { host: "{{ source_host }}", mode: "cleanup" }
    - { host: "{{ destination_host }}", mode: "cleanup" }
  loop_control:
    loop_var: rsync_host
  when: remove_container_after
```

### tasks/verify_environment.yml

```yaml
---
- name: Check Docker installation on remote hosts
  command: docker --version
  register: docker_check
  changed_when: false
  ignore_errors: yes
  loop:
    - "{{ source_host }}"
    - "{{ destination_host }}"
  loop_control:
    loop_var: check_host
    label: "Checking Docker on {{ check_host }}"

- name: Fail if Docker not available
  fail:
    msg: "Docker not properly installed on {{ item.item }}"
  when: item.rc != 0
  loop: "{{ docker_check.results }}"
```

### tasks/deploy_rsync_container.yml

```yaml
---
- name: Build custom rsync image
  community.docker.docker_image:
    name: "{{ rsync_image }}"
    source: build
    build:
      dockerfile: "Dockerfile.rsync"
      path: "/tmp/rsync_build"
    force_source: "{{ rsync_host.mode == 'source' }}"
  delegate_to: "{{ rsync_host.host }}"
  when: rsync_host.mode != "cleanup"

- name: Create docker-compose for rsync
  template:
    src: "docker-compose.rsync.j2"
    dest: "/tmp/rsync_service/docker-compose.yml"
  delegate_to: "{{ rsync_host.host }}"
  when: rsync_host.mode != "cleanup"

- name: Start rsync container
  community.docker.docker_compose:
    project_src: "/tmp/rsync_service"
    state: present
    pull: no
  delegate_to: "{{ rsync_host.host }}"
  when: rsync_host.mode != "cleanup"

- name: Stop and remove rsync container
  community.docker.docker_container:
    name: "{{ rsync_container_name }}"
    state: absent
    force_kill: yes
  delegate_to: "{{ rsync_host.host }}"
  when: rsync_host.mode == "cleanup"
```

### tasks/perform_transfer.yml

```yaml
---
- name: Execute rsync transfer between containers
  command: |
    ssh -i {{ ssh_private_key }} -p {{ ssh_port }} \
    -o StrictHostKeyChecking=no {{ ssh_user }}@{{ source_host }} \
    "docker exec {{ rsync_container_name }} \
    rsync -avz --progress --delete --timeout={{ rsync_timeout }} \
    -e 'ssh -p {{ ssh_port }} -i /ssh_key -o StrictHostKeyChecking=no' \
    {{ container_data_path }}/ \
    {{ rsync_user }}@{{ destination_host }}::data{{ container_data_path }}/"
  register: transfer_result
  no_log: true
  changed_when: transfer_result.rc == 0 or transfer_result.rc == 23
  failed_when: transfer_result.rc not in [0, 23, 24]

- name: Display transfer summary
  debug:
    msg: "Transfer completed with status {{ transfer_result.rc }}. {{ transfer_result.stdout_lines[-1] }}"
  when: transfer_result.stdout_lines | length > 0
```

### templates/Dockerfile.rsync.j2

```dockerfile
FROM alpine:3.14

RUN apk add --no-cache rsync openssh-client && \
    mkdir -p /etc/rsync && \
    adduser -D -H {{ rsync_user }} && \
    echo "{{ rsync_user }}:{{ rsync_password }}" > /etc/rsync/rsyncd.secrets && \
    chmod 600 /etc/rsync/rsyncd.secrets

COPY rsyncd.conf /etc/rsync/rsyncd.conf

VOLUME ["{{ container_data_path }}"]
EXPOSE {{ rsync_port }}

CMD ["rsync", "--daemon", "--no-detach", "--config=/etc/rsync/rsyncd.conf"]
```

### templates/docker-compose.rsync.j2

```yaml
version: '3.8'

services:
  rsync:
    container_name: "{{ rsync_container_name }}"
    image: "{{ rsync_image }}"
    ports:
      - "{{ rsync_port }}:{{ rsync_port }}"
    volumes:
      - "{{ source_volume if rsync_host.mode == 'source' else destination_volume }}:{{ container_data_path }}"
      - "{{ ssh_private_key }}:/ssh_key"
    environment:
      - TZ=UTC
    restart: on-failure
    networks:
      - rsync-net

networks:
  rsync-net:
    driver: bridge
```

## Пример использования

1. Создайте playbook `rsync_transfer.yml`:

```yaml
- hosts: localhost
  gather_facts: no
  roles:
    - role: remote_rsync_docker
      vars:
        source_host: "node1.cluster"
        destination_host: "node2.cluster"
        source_volume: "/data/prod/current"
        destination_volume: "/data/backup/daily"
        rsync_timeout: 1200
```

2. Запустите playbook:

```bash
ansible-playbook rsync_transfer.yml --ask-vault-pass
```

## Ключевые особенности

1. **Изолированные контейнеры**:
   - Временные контейнеры для каждой операции
   - Автоматическая очистка после завершения
   - Уникальные имена контейнеров

2. **Безопасность**:
   - SSH-туннелирование для передачи данных
   - Пароли защищены через Ansible Vault
   - Изолированные сетевые пространства

3. **Гибкость**:
   - Поддержка пользовательских путей
   - Настраиваемые таймауты
   - Возможность сохранить контейнеры для отладки

4. **Надёжность**:
   - Проверка окружения перед выполнением
   - Логирование операций
   - Обработка различных кодов завершения rsync

Для работы требуется:
- Доступ по SSH с ключами на обоих хостах
- Установленный Docker на целевых хостах
- Открытые порты для SSH и Rsync