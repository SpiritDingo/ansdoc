# Ansible роль для переключения репозиториев с Spacewalk на Nexus

Эта роль выполняет миграцию пакетных репозиториев с Spacewalk на Nexus для серверов под управлением Oracle Linux и Ubuntu.

## Структура роли

```
roles/migrate_repos_from_spacewalk/
├── defaults
│   └── main.yml
├── tasks
│   ├── main.yml
│   ├── oraclelinux.yml
│   └── ubuntu.yml
├── templates
│   ├── oraclelinux.repo.j2
│   └── ubuntu.list.j2
└── vars
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки по умолчанию
nexus_url: "http://nexus.example.com"
nexus_repo_port: 8081
nexus_ol_repo_name: "ol-repo"
nexus_ubuntu_repo_name: "ubuntu-repo"
gpg_check: false

# Настройки Spacewalk (для очистки)
spacewalk_client_config: "/etc/sysconfig/rhn/up2date"
```

### vars/main.yml

```yaml
---
# Специфичные переменные для дистрибутивов
ol_versions:
  - "8"
  - "9"

ubuntu_versions:
  - "focal"
  - "jammy"

# Пакеты Spacewalk для удаления
spacewalk_packages:
  oraclelinux:
    - "rhn-client-tools"
    - "rhn-check"
    - "rhn-setup"
    - "yum-rhn-plugin"
  ubuntu:
    - "python-rhn"
    - "rhnsd"
    - "spacewalk-client"
```

### tasks/main.yml

```yaml
---
- name: Detect if Spacewalk is configured
  stat:
    path: "{{ spacewalk_client_config }}"
  register: spacewalk_config
  ignore_errors: yes

- name: Include OS-specific tasks
  include_tasks: "{{ ansible_distribution | lower }}.yml"
  when: 
    - ansible_distribution in ['OracleLinux', 'Ubuntu']
    - spacewalk_config.stat.exists
```

### tasks/oraclelinux.yml

```yaml
---
- name: Remove Spacewalk packages (Oracle Linux)
  become: yes
  yum:
    name: "{{ item }}"
    state: absent
  loop: "{{ spacewalk_packages.oraclelinux }}"
  ignore_errors: yes

- name: Clean up Spacewalk configuration
  become: yes
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - "/etc/sysconfig/rhn"
    - "/etc/yum.repos.d/spacewalk*.repo"

- name: Backup existing repos
  become: yes
  copy:
    src: "/etc/yum.repos.d/"
    dest: "/tmp/yum.repos.d.backup-{{ ansible_date_time.iso8601 }}"
    remote_src: yes

- name: Create Nexus repo configuration for Oracle Linux
  become: yes
  template:
    src: "oraclelinux.repo.j2"
    dest: "/etc/yum.repos.d/nexus-ol{{ ol_version }}.repo"
  loop: "{{ ol_versions }}"
  loop_control:
    loop_var: ol_version

- name: Clean yum cache
  become: yes
  command: yum clean all
```

### tasks/ubuntu.yml

```yaml
---
- name: Remove Spacewalk packages (Ubuntu)
  become: yes
  apt:
    name: "{{ item }}"
    state: absent
  loop: "{{ spacewalk_packages.ubuntu }}"
  ignore_errors: yes

- name: Clean up Spacewalk configuration
  become: yes
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - "/etc/apt/sources.list.d/spacewalk.list"
    - "/etc/apt/auth.conf.d/spacewalk.conf"

- name: Backup existing sources.list
  become: yes
  copy:
    src: "/etc/apt/sources.list"
    dest: "/tmp/sources.list.backup-{{ ansible_date_time.iso8601 }}"
    remote_src: yes

- name: Remove existing .list files
  become: yes
  file:
    path: "/etc/apt/sources.list.d/{{ item }}"
    state: absent
  loop: "{{ lookup('fileglob', '/etc/apt/sources.list.d/*.list', wantlist=True) }}"

- name: Create Nexus repo configuration for Ubuntu
  become: yes
  template:
    src: "ubuntu.list.j2"
    dest: "/etc/apt/sources.list.d/nexus-{{ ubuntu_version }}.list"
  loop: "{{ ubuntu_versions }}"
  loop_control:
    loop_var: ubuntu_version

- name: Update apt cache
  become: yes
  apt:
    update_cache: yes
```

### templates/oraclelinux.repo.j2

```jinja2
[nexus-ol{{ ol_version }}]
name=Nexus Repository for Oracle Linux {{ ol_version }}
baseurl={{ nexus_url }}:{{ nexus_repo_port }}/repository/{{ nexus_ol_repo_name }}/oraclelinux/{{ ol_version }}/
enabled=1
gpgcheck={{ 1 if gpg_check else 0 }}
{% if gpg_check %}
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle
{% endif %}
```

### templates/ubuntu.list.j2

```jinja2
deb [trusted=yes] {{ nexus_url }}:{{ nexus_repo_port }}/repository/{{ nexus_ubuntu_repo_name }}/ {{ ubuntu_version }} main restricted universe multiverse
deb [trusted=yes] {{ nexus_url }}:{{ nexus_repo_port }}/repository/{{ nexus_ubuntu_repo_name }}/ {{ ubuntu_version }}-updates main restricted universe multiverse
deb [trusted=yes] {{ nexus_url }}:{{ nexus_repo_port }}/repository/{{ nexus_ubuntu_repo_name }}/ {{ ubuntu_version }}-security main restricted universe multiverse
```

## Пример использования роли

```yaml
- hosts: all
  roles:
    - role: migrate_repos_from_spacewalk
      vars:
        nexus_url: "http://nexus.internal.company.com"
        nexus_repo_port: 8081
        nexus_ol_repo_name: "ol-repo"
        nexus_ubuntu_repo_name: "ubuntu-repo"
        gpg_check: false
```

## Особенности роли

1. Автоматически определяет наличие конфигурации Spacewalk
2. Удаляет клиентские пакеты Spacewalk
3. Очищает конфигурационные файлы Spacewalk
4. Настраивает репозитории Nexus для замены Spacewalk
5. Поддерживает несколько версий Oracle Linux и Ubuntu
6. Создает резервные копии существующей конфигурации

Для работы роли необходимо:
1. Предварительно настроить репозитории в Nexus
2. Убедиться, что серверы имеют доступ к Nexus-серверу
3. Проверить совместимость версий пакетов между Spacewalk и Nexus