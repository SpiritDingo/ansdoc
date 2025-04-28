# Перенос Docker образов и развертывание на новом сервере с помощью Ansible

Для переноса Docker образов с одного сервера на другой и развертывания с помощью Docker Compose через Ansible можно создать следующую роль:

## Структура роли

```
roles/docker_migration/
├── tasks/
│   ├── main.yml
│   ├── save_images.yml
│   ├── transfer_images.yml
│   ├── load_images.yml
│   └── deploy_compose.yml
├── defaults/
│   └── main.yml
└── templates/
    └── docker-compose.yml.j2
```

## Файлы роли

### defaults/main.yml

```yaml
# Исходный сервер (откуда переносим)
source_host: old_server
source_images:
  - nginx:latest
  - postgres:13
  - my_custom_image:1.0

# Целевой сервер (куда переносим)
target_host: new_server

# Пути для временных файлов
temp_dir: /tmp/docker_migration
compose_project_dir: /opt/my_app
```

### tasks/main.yml

```yaml
- name: Include all migration tasks
  include_tasks: save_images.yml
  when: inventory_hostname == source_host

- name: Transfer and load images
  include_tasks: transfer_images.yml
  when: inventory_hostname == target_host

- name: Deploy docker compose
  include_tasks: deploy_compose.yml
  when: inventory_hostname == target_host
```

### tasks/save_images.yml

```yaml
- name: Ensure temp directory exists on source
  file:
    path: "{{ temp_dir }}"
    state: directory
    mode: '0755'

- name: Save Docker images to tar files
  command: "docker save -o {{ temp_dir }}/{{ item | replace('/', '_') | replace(':', '_') }}.tar {{ item }}"
  loop: "{{ source_images }}"
  register: save_result
  changed_when: "'No such image' not in save_result.stderr"

- name: Show save results
  debug:
    var: save_result
```

### tasks/transfer_images.yml

```yaml
- name: Ensure temp directory exists on target
  file:
    path: "{{ temp_dir }}"
    state: directory
    mode: '0755'

- name: Transfer image files from source to target
  synchronize:
    src: "{{ temp_dir }}/"
    dest: "{{ temp_dir }}"
    mode: pull
    delegate_to: "{{ source_host }}"
```

### tasks/load_images.yml

```yaml
- name: Load Docker images from tar files
  command: "docker load -i {{ temp_dir }}/{{ item | replace('/', '_') | replace(':', '_') }}.tar"
  loop: "{{ source_images }}"
  register: load_result
  changed_when: "'Loaded image' in load_result.stdout"

- name: Show load results
  debug:
    var: load_result

- name: Cleanup temp files
  file:
    path: "{{ temp_dir }}"
    state: absent
```

### tasks/deploy_compose.yml

```yaml
- name: Install Docker
  apt:
    name:
      - docker.io
      - docker-compose
    state: present
    update_cache: yes

- name: Ensure project directory exists
  file:
    path: "{{ compose_project_dir }}"
    state: directory
    mode: '0755'

- name: Deploy docker-compose template
  template:
    src: docker-compose.yml.j2
    dest: "{{ compose_project_dir }}/docker-compose.yml"

- name: Start containers with docker-compose
  command: docker-compose up -d
  args:
    chdir: "{{ compose_project_dir }}"
```

### templates/docker-compose.yml.j2

```yaml
version: '3'

services:
  web:
    image: nginx:latest
    ports:
      - "80:80"
  
  db:
    image: postgres:13
    environment:
      POSTGRES_PASSWORD: example
    volumes:
      - pg_data:/var/lib/postgresql/data

  custom_app:
    image: my_custom_image:1.0
    depends_on:
      - db

volumes:
  pg_data:
```

## Playbook для использования роли

Создайте файл `migrate_docker.yml`:

```yaml
- hosts: "{{ source_host }}"
  roles:
    - docker_migration

- hosts: "{{ target_host }}"
  roles:
    - docker_migration
```

## Использование

1. Запустите playbook:
   ```bash
   ansible-playbook migrate_docker.yml
   ```

2. Для указания конкретных образов можно переопределить переменную:
   ```bash
   ansible-playbook migrate_docker.yml -e "source_images=['nginx:latest','custom:image']"
   ```

## Дополнительные улучшения

1. Добавьте обработку ошибок и проверку существования образов
2. Реализуйте резервное копирование перед миграцией
3. Добавьте теги для разделения задач (сохранение, перенос, развертывание)
4. Реализуйте проверку версий Docker на обоих серверах

Эта роль обеспечит полный цикл переноса Docker образов и развертывания сервисов с помощью Docker Compose на новом сервере.