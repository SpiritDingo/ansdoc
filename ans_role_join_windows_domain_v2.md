# Ansible роль для ввода Oracle Linux 9 в Windows домен

Ниже представлена Ansible роль, которая автоматизирует процесс присоединения Oracle Linux 9 к Windows Active Directory домену.

## Структура роли

```
roles/join_windows_domain/
├── tasks/
│   └── main.yml
├── defaults/
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
# Имя домена (например, example.com)
domain_name: "example.com"

# Имя домена в формате NETBIOS (например, EXAMPLE)
domain_netbios_name: "EXAMPLE"

# Пользователь домена с правами на добавление компьютеров
domain_admin_user: "admin"

# Пароль доменного администратора (должен быть установлен через vault)
domain_admin_password: ""

# OU, в которую будет помещен компьютер (опционально)
domain_ou: ""

# Имя компьютера в домене (если пусто - будет использовано текущее имя хоста)
domain_computer_name: ""

# Настройки SSSD
sssd_backend: "ad"
sssd_services: "nss, pam"
```

### tasks/main.yml

```yaml
---
- name: Установка необходимых пакетов
  dnf:
    name:
      - sssd
      - adcli
      - samba-common-tools
      - krb5-workstation
      - oddjob
      - oddjob-mkhomedir
      - authselect
      - authselect-compat
    state: present

- name: Настройка файла /etc/krb5.conf
  template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
    owner: root
    group: root
    mode: 0644

- name: Присоединение к домену с помощью adcli
  command: >
    adcli join {{ domain_name }} 
    --login-user="{{ domain_admin_user }}" 
    --login-password="{{ domain_admin_password }}" 
    --domain-ou="{{ domain_ou }}" 
    --computer-name="{{ domain_computer_name | default(omit) }}"
  register: join_result
  changed_when: "'Already joined to domain' not in join_result.stderr"
  no_log: true

- name: Настройка authselect
  command: >
    authselect select sssd 
    with-mkhomedir 
    with-sudo 
    with-smartcard 
    --force

- name: Настройка SSSD
  ini_file:
    path: /etc/sssd/sssd.conf
    section: sssd
    option: "{{ item.key }}"
    value: "{{ item.value }}"
    mode: 0600
    owner: root
    group: root
  with_dict:
    config_file_version: 2
    services: "{{ sssd_services }}"
    domains: "{{ domain_name }}"
    default_domain_suffix: "{{ domain_name }}"

- name: Настройка домена в SSSD
  ini_file:
    path: /etc/sssd/sssd.conf
    section: "domain/{{ domain_name }}"
    option: "{{ item.key }}"
    value: "{{ item.value }}"
  with_dict:
    id_provider: "{{ sssd_backend }}"
    auth_provider: ad
    chpass_provider: ad
    access_provider: ad
    ad_domain: "{{ domain_name }}"
    ad_server: "{{ domain_name }}"
    krb5_realm: "{{ domain_name | upper }}"
    krb5_store_password_if_offline: True
    cache_credentials: True
    default_shell: /bin/bash
    ldap_id_mapping: True
    fallback_homedir: /home/%u
    use_fully_qualified_names: False

- name: Включение и запуск службы SSSD
  service:
    name: sssd
    state: started
    enabled: yes

- name: Настройка sudo для доменных администраторов
  lineinfile:
    path: /etc/sudoers.d/domain_admins
    line: '%Domain\ Admins@{{ domain_netbios_name }} ALL=(ALL) ALL'
    create: yes
    validate: 'visudo -cf %s'

- name: Перезапуск службы SSSD для применения изменений
  service:
    name: sssd
    state: restarted
```

### templates/krb5.conf.j2

```jinja2
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 default_realm = {{ domain_name | upper }}
 default_ccache_name = KEYRING:persistent:%{uid}

[realms]
 {{ domain_name | upper }} = {
  kdc = {{ domain_name }}
  admin_server = {{ domain_name }}
 }

[domain_realm]
 .{{ domain_name }} = {{ domain_name | upper }}
 {{ domain_name }} = {{ domain_name | upper }}
```

## Использование роли

1. Создайте playbook (например, `join_domain.yml`):

```yaml
---
- hosts: all
  become: yes
  roles:
    - join_windows_domain
  vars:
    domain_name: "yourdomain.com"
    domain_netbios_name: "YOURDOMAIN"
    domain_admin_user: "joinuser"
    domain_admin_password: "securepassword"
    domain_ou: "OU=Linux Servers,DC=yourdomain,DC=com"
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory join_domain.yml --ask-become-pass
```

## Важные замечания

1. Пароль доменного администратора должен храниться безопасно (используйте Ansible Vault).
2. Убедитесь, что DNS серверы правильно настроены на клиенте и указывают на контроллер домена.
3. Время на сервере Oracle Linux должно быть синхронизировано с доменом (рекомендуется использовать NTP).
4. Для работы требуется открытый доступ к портам доменных служб (обычно TCP/UDP 88, 389, 445, 464, 3268 и другие).

Эта роль выполняет базовую настройку для присоединения к домену. В зависимости от ваших требований, может потребоваться дополнительная настройка SSSD или PAM.