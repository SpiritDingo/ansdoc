# Ansible роль для настройки репозиториев на Oracle Linux и Ubuntu

Эта роль настраивает пакетные репозитории на удалённых серверах с Oracle Linux и Ubuntu, используя Nexus в качестве источника.

## Структура роли

```
roles/configure_repos/
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
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific tasks
  include_tasks: "{{ ansible_distribution | lower }}.yml"
  when: ansible_distribution in ['OracleLinux', 'Ubuntu']
```

### tasks/oraclelinux.yml

```yaml
---
- name: Backup existing repos
  become: yes
  copy:
    src: "/etc/yum.repos.d/"
    dest: "/tmp/yum.repos.d.backup-{{ ansible_date_time.iso8601 }}"
    remote_src: yes

- name: Remove existing repos
  become: yes
  file:
    path: "/etc/yum.repos.d/{{ item }}"
    state: absent
  loop: "{{ lookup('fileglob', '/etc/yum.repos.d/*.repo', wantlist=True) }}"

- name: Create Nexus repo configuration for Oracle Linux
  become: yes
  template:
    src: "oraclelinux.repo.j2"
    dest: "/etc/yum.repos.d/nexus-ol{{ ol_version }}.repo"
  loop: "{{ ol_versions }}"
  loop_control:
    loop_var: ol_version
```

### tasks/ubuntu.yml

```yaml
---
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
    - role: configure_repos
      vars:
        nexus_url: "http://nexus.internal.company.com"
        nexus_repo_port: 8081
        nexus_ol_repo_name: "ol-repo"
        nexus_ubuntu_repo_name: "ubuntu-repo"
        gpg_check: false
```

## Особенности роли

1. Поддерживает несколько версий Oracle Linux и Ubuntu
2. Создаёт резервные копии существующих репозиториев перед изменениями
3. Позволяет включать/выключать проверку GPG
4. Использует шаблоны для генерации конфигурационных файлов
5. Автоматически определяет ОС и применяет соответствующие задачи

Для работы роли необходимо предварительно настроить репозитории в Nexus и убедиться, что серверы имеют доступ к Nexus-серверу.