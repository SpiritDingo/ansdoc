# Ansible роль для управления пользовательскими репозиториями

Ниже представлена Ansible роль для добавления и удаления пользовательских репозиториев на Linux системах (поддерживающих apt или yum/dnf).

## Структура роли

```
roles/custom_repository/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
│   ├── apt-repo.j2
│   └── yum-repo.j2
└── README.md
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки репозитория по умолчанию
custom_repositories:
  - name: "my-custom-repo"
    state: "present"  # или "absent" для удаления
    description: "My Custom Repository"
    # Для APT
    apt:
      url: "http://example.com/repo"
      distribution: "focal"
      components: "main"
      arch: "amd64"
      key_url: "http://example.com/repo/key.gpg"
    # Для YUM/DNF
    yum:
      baseurl: "http://example.com/repo"
      gpgkey: "http://example.com/repo/key.gpg"
      gpgcheck: yes
      enabled: yes
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution|lower }}-{{ ansible_distribution_version|lower }}.yml"
    - "{{ ansible_distribution|lower }}-{{ ansible_distribution_major_version|lower }}.yml"
    - "{{ ansible_distribution|lower }}.yml"
    - "{{ ansible_os_family|lower }}.yml"
  ignore_errors: yes

- name: Install prerequisites (APT)
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - gnupg
    state: present
  when: ansible_pkg_mgr == 'apt'

- name: Add APT repository key
  apt_key:
    url: "{{ item.apt.key_url }}"
    state: present
  loop: "{{ custom_repositories }}"
  when:
    - ansible_pkg_mgr == 'apt'
    - item.state == 'present'
    - item.apt.key_url is defined

- name: Add APT repository
  template:
    src: apt-repo.j2
    dest: "/etc/apt/sources.list.d/{{ item.name }}.list"
    owner: root
    group: root
    mode: '0644'
  loop: "{{ custom_repositories }}"
  when:
    - ansible_pkg_mgr == 'apt'
    - item.state == 'present'
  notify: Update APT cache

- name: Remove APT repository
  file:
    path: "/etc/apt/sources.list.d/{{ item.name }}.list"
    state: absent
  loop: "{{ custom_repositories }}"
  when:
    - ansible_pkg_mgr == 'apt'
    - item.state == 'absent'

- name: Add YUM/DNF repository
  template:
    src: yum-repo.j2
    dest: "/etc/yum.repos.d/{{ item.name }}.repo"
    owner: root
    group: root
    mode: '0644'
  loop: "{{ custom_repositories }}"
  when:
    - ansible_pkg_mgr in ['yum', 'dnf']
    - item.state == 'present'

- name: Remove YUM/DNF repository
  file:
    path: "/etc/yum.repos.d/{{ item.name }}.repo"
    state: absent
  loop: "{{ custom_repositories }}"
  when:
    - ansible_pkg_mgr in ['yum', 'dnf']
    - item.state == 'absent'

- name: Clean YUM/DNF cache
  command: "yum clean all"
  when:
    - ansible_pkg_mgr in ['yum', 'dnf']
    - custom_repositories | selectattr('state', 'equalto', 'absent') | list | length > 0
```

### handlers/main.yml

```yaml
---
- name: Update APT cache
  apt:
    update_cache: yes
```

### templates/apt-repo.j2

```
# {{ item.description }}
deb [arch={{ item.apt.arch }}] {{ item.apt.url }} {{ item.apt.distribution }} {{ item.apt.components }}
```

### templates/yum-repo.j2

```
[{{ item.name }}]
name={{ item.description }}
baseurl={{ item.yum.baseurl }}
{% if item.yum.gpgkey is defined %}
gpgkey={{ item.yum.gpgkey }}
{% endif %}
gpgcheck={{ 1 if item.yum.gpgcheck else 0 }}
enabled={{ 1 if item.yum.enabled else 0 }}
```

## Пример использования

1. Создайте playbook `manage_repositories.yml`:

```yaml
---
- hosts: all
  become: yes
  roles:
    - custom_repository
  vars:
    custom_repositories:
      - name: "my-awesome-repo"
        state: "present"
        description: "My Awesome Software Repository"
        apt:
          url: "https://apt.example.com"
          distribution: "focal"
          components: "main"
          arch: "amd64"
          key_url: "https://apt.example.com/key.gpg"
        yum:
          baseurl: "https://yum.example.com"
          gpgkey: "https://yum.example.com/key.gpg"
          gpgcheck: yes
          enabled: yes
```

2. Для добавления репозитория выполните:
```bash
ansible-playbook -i inventory manage_repositories.yml
```

3. Для удаления репозитория измените `state: "absent"` и снова запустите playbook.

## Особенности

- Роль автоматически определяет менеджер пакетов (APT или YUM/DNF)
- Поддерживает добавление GPG-ключей для репозиториев
- Обновляет кэш пакетов после изменений
- Может управлять несколькими репозиториями одновременно
- Имеет понятные параметры конфигурации

Вы можете расширить эту роль, добавив поддержку других дистрибутивов или дополнительные параметры репозиториев.