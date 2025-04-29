# Ansible роль для настройки и добавления Linux серверов в Active Directory

Ниже представлена структура Ansible роли для интеграции Linux серверов (как DEB, так и RPM-based) в домен Active Directory.

## Структура роли

```
roles/ad_integration/
├── defaults
│   └── main.yml       # Основные переменные по умолчанию
├── tasks
│   ├── main.yml       # Основные задачи
│   ├── deb.yml        # Задачи для Debian/Ubuntu
│   ├── rpm.yml        # Задачи для RHEL/CentOS
│   └── common.yml     # Общие задачи для всех дистрибутивов
├── templates
│   ├── krb5.conf.j2   # Шаблон конфигурации Kerberos
│   ├── sssd.conf.j2   # Шаблон конфигурации SSSD
│   └── smb.conf.j2    # Шаблон конфигурации Samba (опционально)
└── vars
    └── main.yml       # Специфичные переменные
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки по умолчанию для интеграции с AD
ad_integration:
  enabled: true
  domain: "example.com"
  domain_controllers:
    - "dc1.example.com"
    - "dc2.example.com"
  domain_admin_user: "joinuser"
  domain_admin_password: ""  # Должен быть переопределен в vault или переменных
  ou: "OU=Linux Servers,DC=example,DC=com"
  sssd:
    config:
      debug_level: "0x02F0"
      default_shell: "/bin/bash"
      default_homedir: "/home/%u"
      fallback_homedir: "/home/%u"
      krb5_store_password_if_offline: "True"
      use_fully_qualified_names: "False"
  krb5:
    config:
      default_realm: "EXAMPLE.COM"
      ticket_lifetime: "24h"
      renew_lifetime: "7d"
      forwardable: "true"
  packages:
    deb:
      - "realmd"
      - "sssd"
      - "sssd-tools"
      - "libnss-sss"
      - "libpam-sss"
      - "adcli"
      - "samba-common-bin"
      - "krb5-user"
    rpm:
      - "realmd"
      - "sssd"
      - "adcli"
      - "samba-common-tools"
      - "oddjob"
      - "oddjob-mkhomedir"
      - "krb5-workstation"
```

### tasks/main.yml

```yaml
---
- name: Include distribution-specific tasks
  block:
    - name: Include Debian/Ubuntu tasks
      ansible.builtin.include_tasks: deb.yml
      when: ansible_facts['os_family'] == 'Debian'

    - name: Include RHEL/CentOS tasks
      ansible.builtin.include_tasks: rpm.yml
      when: ansible_facts['os_family'] == 'RedHat'
  
  when: ad_integration.enabled

- name: Include common tasks for AD integration
  ansible.builtin.include_tasks: common.yml
  when: ad_integration.enabled
```

### tasks/deb.yml

```yaml
---
- name: Install required packages for Debian/Ubuntu
  ansible.builtin.apt:
    name: "{{ ad_integration.packages.deb }}"
    state: present
    update_cache: yes

- name: Configure Kerberos (Debian/Ubuntu)
  ansible.builtin.template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
    owner: root
    group: root
    mode: '0644'
  notify: Restart SSSD
```

### tasks/rpm.yml

```yaml
---
- name: Install required packages for RHEL/CentOS
  ansible.builtin.yum:
    name: "{{ ad_integration.packages.rpm }}"
    state: present

- name: Configure Kerberos (RHEL/CentOS)
  ansible.builtin.template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
    owner: root
    group: root
    mode: '0644'
  notify: Restart SSSD

- name: Enable and start oddjobd service (RHEL/CentOS)
  ansible.builtin.systemd:
    name: oddjobd
    enabled: yes
    state: started
```

### tasks/common.yml

```yaml
---
- name: Discover AD domain
  ansible.builtin.command: realm discover {{ ad_integration.domain }}
  register: realm_discover
  changed_when: false

- name: Join to AD domain
  ansible.builtin.command: >
    realm join -v
    -U "{{ ad_integration.domain_admin_user }}"
    --computer-ou="{{ ad_integration.ou }}"
    {{ ad_integration.domain }}
  args:
    stdin: "{{ ad_integration.domain_admin_password }}\n"
  register: realm_join
  changed_when: realm_join.rc == 0
  no_log: true

- name: Configure SSSD
  ansible.builtin.template:
    src: sssd.conf.j2
    dest: /etc/sssd/sssd.conf
    owner: root
    group: root
    mode: '0600'
  notify: Restart SSSD

- name: Configure Samba (optional)
  ansible.builtin.template:
    src: smb.conf.j2
    dest: /etc/samba/smb.conf
    owner: root
    group: root
    mode: '0644'
  when: ad_integration.samba.enabled | default(false)

- name: Enable and start SSSD service
  ansible.builtin.systemd:
    name: sssd
    enabled: yes
    state: started

- name: Configure PAM for homedir creation
  ansible.builtin.lineinfile:
    path: /etc/pam.d/common-session
    line: "session required pam_mkhomedir.so skel=/etc/skel umask=0022"
    state: present
  when: ansible_facts['os_family'] == 'Debian'

- name: Configure sudoers for AD groups
  ansible.builtin.blockinfile:
    path: /etc/sudoers.d/ad_admins
    block: |
      %Domain\ Admins@{{ ad_integration.domain }} ALL=(ALL) ALL
      %Linux\ Admins@{{ ad_integration.domain }} ALL=(ALL) ALL
    create: yes
    mode: '0440'
    validate: 'visudo -cf %s'
```

### templates/krb5.conf.j2

```
[libdefaults]
    default_realm = {{ ad_integration.krb5.config.default_realm | upper }}
    dns_lookup_realm = true
    dns_lookup_kdc = true
    ticket_lifetime = {{ ad_integration.krb5.config.ticket_lifetime }}
    renew_lifetime = {{ ad_integration.krb5.config.renew_lifetime }}
    forwardable = {{ ad_integration.krb5.config.forwardable }}
    rdns = false
    default_ccache_name = KEYRING:persistent:%{uid}

[realms]
    {{ ad_integration.krb5.config.default_realm | upper }} = {
        kdc = {{ ad_integration.domain_controllers | first }}
        admin_server = {{ ad_integration.domain_controllers | first }}
    }

[domain_realm]
    .{{ ad_integration.domain | lower }} = {{ ad_integration.krb5.config.default_realm | upper }}
    {{ ad_integration.domain | lower }} = {{ ad_integration.krb5.config.default_realm | upper }}
```

### templates/sssd.conf.j2

```
[sssd]
domains = {{ ad_integration.domain }}
config_file_version = 2
services = nss, pam
debug_level = {{ ad_integration.sssd.config.debug_level }}

[domain/{{ ad_integration.domain }}]
id_provider = ad
access_provider = ad
auth_provider = ad
chpass_provider = ad

# AD settings
ad_domain = {{ ad_integration.domain }}
ad_server = {{ ad_integration.domain_controllers | join(',') }}
ad_hostname = {{ ansible_hostname }}.{{ ad_integration.domain }}

# User settings
default_shell = {{ ad_integration.sssd.config.default_shell }}
default_homedir = {{ ad_integration.sssd.config.default_homedir }}
fallback_homedir = {{ ad_integration.sssd.config.fallback_homedir }}
use_fully_qualified_names = {{ ad_integration.sssd.config.use_fully_qualified_names }}

# Kerberos settings
krb5_store_password_if_offline = {{ ad_integration.sssd.config.krb5_store_password_if_offline }}
cache_credentials = True

# LDAP settings
ldap_id_mapping = True
ldap_schema = ad
ldap_user_object_class = user
ldap_group_object_class = group
```

### handlers/main.yml

```yaml
---
- name: Restart SSSD
  ansible.builtin.systemd:
    name: sssd
    state: restarted

- name: Restart Samba
  ansible.builtin.systemd:
    name: smbd
    state: restarted
  when: ad_integration.samba.enabled | default(false)
```

## Пример использования роли

1. Создайте playbook `join_ad.yml`:

```yaml
---
- hosts: linux_servers
  become: yes
  vars_files:
    - vars/ad_vault.yml  # Здесь храните пароли в зашифрованном виде
  roles:
    - ad_integration
  vars:
    ad_integration:
      domain: "example.com"
      domain_controllers:
        - "dc1.example.com"
        - "dc2.example.com"
      domain_admin_user: "joinuser"
      ou: "OU=Linux Servers,DC=example,DC=com"
```

2. Зашифруйте пароль с помощью ansible-vault:

```bash
ansible-vault create vars/ad_vault.yml
```

Содержимое файла:
```yaml
---
ad_integration:
  domain_admin_password: "your_secure_password"
```

3. Запустите playbook:

```bash
ansible-playbook -i inventory.ini --ask-vault-pass join_ad.yml
```

## Дополнительные возможности

1. **Автоматическое обновление ключей Kerberos**:
   - Добавьте cron-задачу для регулярного обновления тикетов

2. **Настройка автоматического монтирования домашних директорий**:
   - Интеграция с autofs для монтирования домашних директорий из CIFS

3. **Настройка политик безопасности**:
   - Конфигурация pam_faillock для защиты от bruteforce
   - Настройка ограничений sudo

4. **Мониторинг интеграции**:
   - Проверка статуса доменного соединения
   - Проверка работоспособности SSSD

5. **Обработка ошибок**:
   - Проверка перед присоединением к домену
   - Повторные попытки при временных сбоях

Эта роль предоставляет комплексное решение для интеграции Linux серверов в Active Directory с поддержкой основных дистрибутивов.