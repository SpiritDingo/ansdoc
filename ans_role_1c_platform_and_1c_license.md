# Установка и настройка 1С:Предприятия и сервера лицензий на Linux с помощью Ansible

## Роль для установки 1С:Предприятия

Создайте структуру роли:
```
roles/1c_server/
├── tasks/
│   ├── main.yml
│   ├── install_platform.yml
│   └── configure_platform.yml
├── templates/
│   └── 1c_config.j2
└── vars/
    └── main.yml
```

### tasks/main.yml
```yaml
- name: Include platform installation tasks
  include: install_platform.yml

- name: Include platform configuration tasks
  include: configure_platform.yml
```

### tasks/install_platform.yml
```yaml
- name: Add 1C repository key
  apt_key:
    url: "http://deb.1c.ru/1C_public.gpg"
    state: present
  when: ansible_os_family == 'Debian'

- name: Add 1C repository (Debian/Ubuntu)
  apt_repository:
    repo: "deb http://deb.1c.ru/{{ 1c_release_name }} {{ ansible_distribution_release }} main"
    state: present
    filename: "1c-enterprise"
  when: ansible_os_family == 'Debian'

- name: Add 1C repository (RHEL/CentOS)
  yum_repository:
    name: "1c-enterprise"
    description: "1C Enterprise repository"
    baseurl: "http://rpm.1c.ru/{{ 1c_release_name }}/{{ ansible_distribution_major_version }}/x86_64"
    gpgcheck: yes
    gpgkey: "http://rpm.1c.ru/1C_public.gpg"
    enabled: yes
  when: ansible_os_family == 'RedHat'

- name: Install 1C Enterprise server
  package:
    name: "{{ 1c_package_name }}"
    state: present
    update_cache: yes

- name: Install additional components
  package:
    name: "{{ item }}"
    state: present
  loop: "{{ 1c_additional_packages }}"
```

### tasks/configure_platform.yml
```yaml
- name: Create 1C service user
  user:
    name: "{{ 1c_service_user }}"
    group: "{{ 1c_service_group }}"
    shell: /bin/bash
    system: yes
    create_home: yes

- name: Create 1C working directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ 1c_service_user }}"
    group: "{{ 1c_service_group }}"
    mode: '0755'
  loop:
    - "{{ 1c_install_dir }}"
    - "{{ 1c_log_dir }}"
    - "{{ 1c_temp_dir }}"

- name: Deploy 1C configuration
  template:
    src: 1c_config.j2
    dest: "{{ 1c_config_path }}"
    owner: "{{ 1c_service_user }}"
    group: "{{ 1c_service_group }}"
    mode: '0644'
  notify: Restart 1C service

- name: Enable and start 1C service
  systemd:
    name: "{{ 1c_service_name }}"
    enabled: yes
    state: started
    daemon_reload: yes
```

### templates/1c_config.j2
```ini
[server]
port = {{ 1c_server_port }}
range = {{ 1c_port_range }}
cluster_port = {{ 1c_cluster_port }}
dbms = {{ 1c_dbms_type }}

[log]
dir = {{ 1c_log_dir }}
level = {{ 1c_log_level }}

[temp]
dir = {{ 1c_temp_dir }}
```

### vars/main.yml
```yaml
1c_release_name: "current"
1c_package_name: "1c-enterprise-server"
1c_additional_packages:
  - "1c-enterprise-server-common"
  - "1c-enterprise-server-nls"
  - "1c-enterprise-ws"
  - "1c-enterprise-common"
  - "1c-enterprise-common-nls"

1c_service_user: "usr1cv8"
1c_service_group: "grp1cv8"

1c_install_dir: "/opt/1C"
1c_log_dir: "/var/log/1C"
1c_temp_dir: "/tmp/1C"

1c_config_path: "/etc/1C/1C.conf"

1c_server_port: 1540
1c_port_range: "1560-1591"
1c_cluster_port: 1541
1c_dbms_type: "postgresql"

1c_log_level: "error"

1c_service_name: "srv1cv83"
```

## Роль для сервера лицензий

Создайте структуру роли:
```
roles/license_server/
├── tasks/
│   ├── main.yml
│   └── install_license_server.yml
└── vars/
    └── main.yml
```

### tasks/main.yml
```yaml
- name: Install and configure 1C license server
  include: install_license_server.yml
```

### tasks/install_license_server.yml
```yaml
- name: Install 1C license server package
  package:
    name: "{{ license_server_package }}"
    state: present

- name: Create license directory
  file:
    path: "{{ license_dir }}"
    state: directory
    owner: "{{ license_service_user }}"
    group: "{{ license_service_group }}"
    mode: '0755'

- name: Copy license files
  copy:
    src: "{{ item }}"
    dest: "{{ license_dir }}"
    owner: "{{ license_service_user }}"
    group: "{{ license_service_group }}"
    mode: '0644'
  loop: "{{ license_files }}"
  when: license_files is defined

- name: Configure license server
  template:
    src: license_server.conf.j2
    dest: "{{ license_config_path }}"
    owner: "{{ license_service_user }}"
    group: "{{ license_service_group }}"
    mode: '0644'
  notify: Restart license server

- name: Enable and start license server
  systemd:
    name: "{{ license_service_name }}"
    enabled: yes
    state: started
```

### vars/main.yml
```yaml
license_server_package: "1c-enterprise-license-server"
license_service_name: "licenseserver"
license_service_user: "usr1cv8"
license_service_group: "grp1cv8"
license_dir: "/var/opt/1C/licenses"
license_config_path: "/etc/1C/1clicense-server.conf"
license_port: 47500
```

## Пример playbook

```yaml
- name: Install and configure 1C Enterprise
  hosts: 1c_servers
  become: yes
  roles:
    - role: 1c_server
      tags: 1c_server

- name: Install and configure 1C License Server
  hosts: license_servers
  become: yes
  roles:
    - role: license_server
      tags: license_server
```

## Дополнительные рекомендации

1. Для работы с PostgreSQL можно добавить отдельную роль или использовать существующую из Ansible Galaxy.
2. Для управления кластером 1С можно использовать модуль `rac` (1C:Enterprise Administration Server Console).
3. Для автоматического добавления лицензий можно использовать скрипты на bash или python, вызываемые через Ansible.
4. Для настройки балансировки нагрузки между несколькими серверами 1С потребуется дополнительная конфигурация.

Пример использования модуля rac:
```yaml
- name: Create 1C cluster
  command: >
    /opt/1C/v8.3/x86_64/rac cluster insert
    --cluster-user={{ 1c_service_user }}
    --cluster-pwd={{ 1c_cluster_password }}
    --cluster-name={{ 1c_cluster_name }}
    --license-server={{ license_server_address }}:{{ license_port }}
  when: inventory_hostname == groups['1c_servers'][0]
```