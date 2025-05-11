# Ansible роль для установки и настройки Nexus Repository CE на Oracle Linux 9 с использованием Docker Compose v2

## Структура роли

```
roles/nexus_ce/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── prerequisites.yml
│   ├── deploy_nexus.yml
│   ├── configure_nexus.yml
│   └── verify_nexus.yml
├── templates/
│   ├── docker-compose.yml.j2
│   └── nexus.properties.j2
├── handlers/
│   └── main.yml
└── vars/
    └── OracleLinux.yml
```

## Содержание файлов

### defaults/main.yml

```yaml
---
# Параметры Nexus
nexus_version: "3.61.0"
nexus_hostname: "nexus.example.com"
nexus_port: 8081
nexus_docker_port: 5000
nexus_data_dir: "/opt/nexus-data"
nexus_admin_password: "admin123"  # В продакшене используйте vault!

# Параметры Docker
docker_compose_version: "v2"
nexus_user: "nexus"
nexus_uid: 200
nexus_gid: 200

# Параметры системы
firewall_configure: true
selinux_configure: true
```

### vars/OracleLinux.yml

```yaml
---
# Специфичные настройки для Oracle Linux 9
required_packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-buildx-plugin
  - docker-compose-plugin
  - git
  - jq
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_distribution }}.yml"

- name: Install prerequisites
  include_tasks: prerequisites.yml
  tags: prerequisites

- name: Deploy Nexus CE
  include_tasks: deploy_nexus.yml
  tags: deploy

- name: Configure Nexus
  include_tasks: configure_nexus.yml
  tags: configure

- name: Verify Nexus installation
  include_tasks: verify_nexus.yml
  tags: verify
```

### tasks/prerequisites.yml

```yaml
---
- name: Install required packages
  dnf:
    name: "{{ required_packages }}"
    state: present
    update_cache: yes

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

- name: Create Nexus user
  user:
    name: "{{ nexus_user }}"
    uid: "{{ nexus_uid }}"
    group: "{{ nexus_user }}"
    system: yes
    shell: /sbin/nologin

- name: Create data directory
  file:
    path: "{{ nexus_data_dir }}"
    state: directory
    owner: "{{ nexus_user }}"
    group: "{{ nexus_user }}"
    mode: '0755'

- name: Configure SELinux for Nexus
  seboolean:
    name: container_manage_cgroup
    state: yes
    persistent: yes
  when: selinux_configure | bool and ansible_selinux.status == "enabled"
```

### tasks/deploy_nexus.yml

```yaml
---
- name: Create Nexus docker-compose directory
  file:
    path: /opt/nexus-compose
    state: directory
    mode: '0755'

- name: Deploy Nexus docker-compose template
  template:
    src: docker-compose.yml.j2
    dest: /opt/nexus-compose/docker-compose.yml
    owner: root
    group: root
    mode: '0644'

- name: Deploy Nexus properties template
  template:
    src: nexus.properties.j2
    dest: "{{ nexus_data_dir }}/etc/nexus.properties"
    owner: "{{ nexus_user }}"
    group: "{{ nexus_user }}"
    mode: '0640'

- name: Pull Nexus image
  command: docker compose -f /opt/nexus-compose/docker-compose.yml pull
  args:
    chdir: /opt/nexus-compose

- name: Start Nexus container
  command: docker compose -f /opt/nexus-compose/docker-compose.yml up -d
  args:
    chdir: /opt/nexus-compose
  register: nexus_start
  notify:
    - wait for nexus

- name: Configure firewall
  firewalld:
    port: "{{ nexus_port }}/tcp"
    permanent: yes
    state: enabled
    immediate: yes
  when: firewall_configure | bool
```

### tasks/configure_nexus.yml

```yaml
---
- name: Wait for Nexus to be ready
  uri:
    url: "http://localhost:{{ nexus_port }}/service/rest/v1/status"
    method: GET
    status_code: 200
    timeout: 30
  register: nexus_status
  until: nexus_status.status == 200
  retries: 30
  delay: 10
  ignore_errors: yes

- name: Change admin password
  uri:
    url: "http://localhost:{{ nexus_port }}/service/rest/v1/security/users/admin/change-password"
    method: PUT
    body: "{{ nexus_admin_password }}"
    body_format: raw
    headers:
      Content-Type: "text/plain"
    status_code: 204
    user: admin
    password: admin123
    force_basic_auth: yes
  when: nexus_status.status == 200
  register: password_change
  retries: 5
  delay: 10
  until: password_change is succeeded
```

### tasks/verify_nexus.yml

```yaml
---
- name: Verify Nexus service
  uri:
    url: "http://localhost:{{ nexus_port }}"
    method: GET
    status_code: 200
    timeout: 10
  register: nexus_verify

- name: Show Nexus status
  debug:
    msg: "Nexus доступен по адресу http://{{ ansible_host }}:{{ nexus_port }}"
  when: nexus_verify.status == 200
```

### templates/docker-compose.yml.j2

```yaml
version: '3.8'

services:
  nexus:
    image: sonatype/nexus3:{{ nexus_version }}
    container_name: nexus
    restart: unless-stopped
    user: "{{ nexus_uid }}:{{ nexus_gid }}"
    volumes:
      - {{ nexus_data_dir }}:/nexus-data
    ports:
      - "{{ nexus_port }}:8081"
      - "{{ nexus_docker_port }}:5000"
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2g -Xmx2g -XX:MaxDirectMemorySize=2g
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081"]
      interval: 30s
      timeout: 10s
      retries: 5
```

### templates/nexus.properties.j2

```properties
# Nexus configuration
application-port={{ nexus_port }}
application-host=0.0.0.0
nexus-args=${jetty.etc}/jetty.xml,${jetty.etc}/jetty-http.xml,${jetty.etc}/jetty-requestlog.xml
nexus-context-path=/

# Nexus security
nexus.scripts.allowCreation=true
```

### handlers/main.yml

```yaml
---
- name: restart docker
  service:
    name: docker
    state: restarted

- name: wait for nexus
  wait_for:
    port: "{{ nexus_port }}"
    timeout: 300
```

## Пример playbook

```yaml
---
- name: Установка и настройка Nexus CE на Oracle Linux 9
  hosts: nexus_servers
  become: yes
  vars:
    nexus_version: "3.62.0"
    nexus_hostname: "nexus.company.local"
    nexus_port: 8081
    nexus_docker_port: 5001
    nexus_data_dir: "/data/nexus"
    nexus_admin_password: "{{ vault_nexus_admin_password }}"

  pre_tasks:
    - name: Проверка Oracle Linux 9
      fail:
        msg: "Этот playbook работает только на Oracle Linux 9"
      when: ansible_distribution != "OracleLinux" or ansible_distribution_major_version != "9"

  roles:
    - role: nexus_ce
      tags: nexus

  post_tasks:
    - name: Проверка версии Nexus
      uri:
        url: "http://localhost:{{ nexus_port }}/service/rest/v1/status"
        method: GET
        return_content: yes
        status_code: 200
      register: nexus_status
      changed_when: false

    - name: Вывод информации о Nexus
      debug:
        msg: "Nexus {{ nexus_version }} успешно установлен. Адрес: http://{{ ansible_host }}:{{ nexus_port }}"
      when: nexus_status.status == 200
```

## Особенности реализации

1. **Использование Docker Compose v2**:
   - Нативная реализация через `docker compose` (не требует отдельной установки)
   - Оптимизировано для Oracle Linux 9

2. **Безопасность**:
   - Отдельный системный пользователь для Nexus
   - Настройка правильных прав на директории
   - Смена пароля admin по умолчанию

3. **Производительность**:
   - Настройка параметров JVM
   - Контроль использования памяти

4. **Отказоустойчивость**:
   - Healthcheck для контейнера
   - Автоматический перезапуск (unless-stopped)

5. **Мониторинг**:
   - Проверка доступности после установки
   - Вывод полезной информации после деплоя

Для применения:
1. Создайте структуру каталогов
2. Заполните файлы как указано
3. Запустите playbook: `ansible-playbook -i inventory deploy_nexus.yml --ask-vault-pass`