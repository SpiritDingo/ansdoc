Я помогу вам создать Ansible роль для установки pgAdmin 4 из локального репозитория Nexus с авторизацией. Вот полная реализация:

Структура роли

```
roles/pgadmin-nexus/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
│   └── pgadmin-nexus.list.j2
├── handlers/
│   └── main.yml
└── vars/
    └── main.yml
```

Файлы роли

defaults/main.yml

```yaml
---
# Default variables for pgadmin-nexus role
pgadmin_version: "latest"
pgadmin_nexus_repo_url: "http://nexus.example.com:8081/repository/ubuntu-pgadmin"
pgadmin_nexus_username: "admin"
pgadmin_nexus_password: "password"
pgadmin_nexus_gpg_key: "http://nexus.example.com:8081/repository/gpg-keys/packages-pgadmin-org.gpg"
pgadmin_install_web: false
pgadmin_install_desktop: true
```

vars/main.yml

```yaml
---
# Variables for pgadmin-nexus role
pgadmin_packages:
  - "pgadmin4-desktop"
  # - "pgadmin4-web"  # Uncomment if web version needed
  # - "pgadmin4-apache2"  # For web version with Apache

pgadmin_apt_preferences: "/etc/apt/preferences.d/pgadmin4"
```

tasks/main.yml

```yaml
---
- name: Install required packages
  apt:
    name:
      - curl
      - gnupg
      - ca-certificates
      - apt-transport-https
    state: present
    update_cache: yes

- name: Create GPG key directory
  file:
    path: /usr/share/keyrings/
    state: directory
    mode: '0755'

- name: Download and install GPG key from Nexus
  get_url:
    url: "{{ pgadmin_nexus_gpg_key }}"
    dest: /usr/share/keyrings/packages-pgadmin-org.gpg
    force: yes
    url_username: "{{ pgadmin_nexus_username }}"
    url_password: "{{ pgadmin_nexus_password }}"
    mode: '0644'
  register: gpg_key_download
  until: gpg_key_download is succeeded
  retries: 3
  delay: 5

- name: Create Nexus repository configuration
  template:
    src: pgadmin-nexus.list.j2
    dest: /etc/apt/sources.list.d/pgadmin-nexus.list
    mode: '0644'
  notify: update apt cache

- name: Create apt preferences for pgadmin
  copy:
    content: |
      Package: pgadmin4*
      Pin: origin {{ pgadmin_nexus_repo_url | regex_replace('^https?://([^/]+).*', '\\1') }}
      Pin-Priority: 1001
    dest: "{{ pgadmin_apt_preferences }}"
    mode: '0644'
  notify: update apt cache

- name: Update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Install pgAdmin 4 packages
  apt:
    name: "{{ item }}"
    state: "{{ 'present' if pgadmin_version == 'latest' else pgadmin_version }}"
    force: yes
  loop: "{{ pgadmin_packages }}"
  notify:
    - restart pgadmin services
    - enable pgadmin services

- name: Verify pgAdmin installation
  command: pgadmin4 --version
  register: pgadmin_version_check
  changed_when: false
  ignore_errors: yes

- name: Display pgAdmin version
  debug:
    msg: "pgAdmin version: {{ pgadmin_version_check.stdout }}"
  when: pgadmin_version_check.rc == 0
```

templates/pgadmin-nexus.list.j2

```jinja2
# pgAdmin 4 repository from Nexus
deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] {{ pgadmin_nexus_repo_url }} {{ ansible_distribution_release }} pgadmin4 main
```

handlers/main.yml

```yaml
---
- name: update apt cache
  apt:
    update_cache: yes
    cache_valid_time: 3600

- name: restart pgadmin services
  systemd:
    name: "{{ item }}"
    state: restarted
    enabled: yes
  loop:
    - "apache2"  # if web version installed
    - "pgadmin4"  # if service exists

- name: enable pgadmin services
  systemd:
    name: "{{ item }}"
    enabled: yes
    state: started
  loop:
    - "apache2"  # if web version installed
```

Playbook для использования роли

playbook.yml

```yaml
---
- name: Install pgAdmin 4 from Nexus repository
  hosts: all
  become: yes
  vars:
    pgadmin_nexus_repo_url: "http://your-nexus-server:8081/repository/ubuntu-pgadmin"
    pgadmin_nexus_username: "{{ vault_nexus_username }}"
    pgadmin_nexus_password: "{{ vault_nexus_password }}"
    pgadmin_nexus_gpg_key: "http://your-nexus-server:8081/repository/gpg-keys/packages-pgadmin-org.gpg"
    pgadmin_install_desktop: true
    pgadmin_install_web: false

  roles:
    - role: pgadmin-nexus
      tags: pgadmin
```

Файл с секретными переменными

group_vars/all/vault.yml (зашифрованный ansible-vault)

```yaml
---
vault_nexus_username: "your-nexus-username"
vault_nexus_password: "your-nexus-password"
```

Использование

1. Создайте роль:

```bash
mkdir -p roles/pgadmin-nexus/{defaults,tasks,templates,handlers,vars}
```

1. Зашифруйте секретные данные:

```bash
ansible-vault create group_vars/all/vault.yml
```

1. Запустите playbook:

```bash
ansible-playbook -i inventory playbook.yml --ask-vault-pass --tags pgadmin
```

Альтернативный вариант с прямым использованием curl (если нужно)

Если вам нужно именно поведение как в вашем примере с curl, добавьте этот task:

```yaml
- name: Download GPG key using curl (alternative method)
  shell: |
    curl -fsS -u {{ pgadmin_nexus_username }}:{{ pgadmin_nexus_password }} \
    "{{ pgadmin_nexus_gpg_key }}" | gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg
  args:
    creates: /usr/share/keyrings/packages-pgadmin-org.gpg
  when: false  # Set to true if you prefer this method
```

Эта роль обеспечивает:

· Установку из локального Nexus репозитория
· Авторизацию при доступе к репозиторию
· Гибкую настройку версий
· Обработку GPG ключей
· Управление сервисами
· Безопасное хранение учетных данных