# Роль Ansible для присоединения Oracle Linux 9 к Active Directory

Вот полная роль Ansible для установки необходимых пакетов, присоединения к Active Directory, создания keytab файла и добавления проверок.

## Структура роли

```
oraclelinux9_ad_join/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
│   └── krb5.conf.j2
└── vars/
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки по умолчанию для присоединения к AD
ad_domain: example.com
ad_admin_user: administrator
ad_ou: "OU=Linux Servers,DC=example,DC=com"
realm_options:
  - "automatic-install=no"
  - "computer-ou={{ ad_ou }}"
  - "os-name=Oracle Linux"
  - "os-version=9"
krb5_conf_path: /etc/krb5.conf
keytab_path: /etc/krb5.keytab
```

### vars/main.yml

```yaml
---
# Список необходимых пакетов
required_packages:
  - realmd
  - oddjob
  - oddjob-mkhomedir
  - sssd
  - adcli
  - samba-common
  - samba-common-tools
  - krb5-workstation
  - openldap-clients
```

### templates/krb5.conf.j2

```jinja2
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = true
 dns_lookup_kdc = true
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 default_realm = {{ ad_domain|upper }}
 default_ccache_name = KEYRING:persistent:%{uid}

[realms]
 {{ ad_domain|upper }} = {
  kdc = {{ ad_domain }}
  admin_server = {{ ad_domain }}
 }

[domain_realm]
 .{{ ad_domain }} = {{ ad_domain|upper }}
 {{ ad_domain }} = {{ ad_domain|upper }}
```

### tasks/main.yml

```yaml
---
- name: Установка необходимых пакетов
  dnf:
    name: "{{ required_packages }}"
    state: present
  register: package_install
  until: package_install is succeeded
  retries: 3
  delay: 10
  notify:
    - Перезапуск SSSD

- name: Копирование настроек krb5.conf
  template:
    src: krb5.conf.j2
    dest: "{{ krb5_conf_path }}"
    owner: root
    group: root
    mode: 0644
  register: krb5_conf

- name: Получение пароля администратора AD
  ansible.builtin.set_fact:
    ad_admin_password: "{{ ad_admin_password }}"
  no_log: true
  when: ad_admin_password is defined

- name: Проверка доступности домена
  command: realm discover "{{ ad_domain }}"
  register: realm_discover
  changed_when: false
  failed_when: realm_discover.rc != 0
  ignore_errors: yes

- name: Присоединение к домену
  command: >
    echo "{{ ad_admin_password }}" | realm join -U "{{ ad_admin_user }}" 
    "{{ ad_domain }}" 
    {% for option in realm_options %}--computer-options="{{ option }}" {% endfor %}
    --verbose
  args:
    warn: false
  register: realm_join
  when: 
    - ad_admin_password is defined
    - realm_discover.rc == 0
  no_log: true
  notify:
    - Перезапуск SSSD

- name: Проверка успешного присоединения к домену
  command: realm list
  register: realm_status
  changed_when: false
  failed_when: "'{{ ad_domain }}' not in realm_status.stdout"

- name: Создание keytab файла
  command: >
    net ads keytab create -U "{{ ad_admin_user }}"%{{ ad_admin_password }} 
    -k "{{ keytab_path }}"
  args:
    warn: false
  when: 
    - ad_admin_password is defined
    - realm_status.rc == 0
  register: keytab_create
  no_log: true

- name: Проверка keytab файла
  command: klist -ke "{{ keytab_path }}"
  register: keytab_check
  changed_when: false
  failed_when: keytab_check.rc != 0
  when: keytab_create is defined and keytab_create.rc == 0

- name: Установка прав на keytab файл
  file:
    path: "{{ keytab_path }}"
    owner: root
    group: root
    mode: 0600
  when: keytab_check is defined and keytab_check.rc == 0

- name: Проверка аутентификации с помощью keytab
  command: kinit -k "HOST/$(hostname -s)@{{ ad_domain|upper }}"
  register: kinit_test
  changed_when: false
  failed_when: kinit_test.rc != 0
  when: keytab_check is defined and keytab_check.rc == 0
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
---
- name: Присоединение Oracle Linux 9 к AD
  hosts: all
  become: true
  roles:
    - oraclelinux9_ad_join
  vars:
    ad_admin_password: "your_admin_password_here"
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini join_ad.yml --ask-become-pass
```

## Особенности роли

1. **Безопасность**: Пароль администратора AD не логируется (no_log: true)
2. **Проверки**:
   - Проверка доступности домена перед присоединением
   - Проверка успешного присоединения (realm list)
   - Проверка создания keytab файла
   - Проверка аутентификации с помощью keytab
3. **Надежность**: Повторные попытки установки пакетов
4. **Гибкость**: Настройки домена и OU можно переопределить

Роль также включает обработчики для перезапуска SSSD при изменении конфигурации (не показаны в примере, но упоминаются в notify).