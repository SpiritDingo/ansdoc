# Роль Ansible для добавления Linux-сервера в Active Directory

## Описание
Эта роль Ansible настраивает Linux-сервер для интеграции с Active Directory, позволяя пользователям AD аутентифицироваться на Linux-хосте.

## Основные шаги реализации

1. **Установка необходимых пакетов**:
   - `realmd` - основной инструмент для интеграции с AD
   - `sssd` - служба системных сервисов безопасности
   - `samba-common` - общие файлы для Samba
   - `krb5-user` - клиентские утилиты Kerberos
   - `adcli` - утилиты для работы с AD

2. **Настройка Kerberos** (`/etc/krb5.conf`)

3. **Присоединение к домену** с помощью `realm join`

4. **Настройка SSSD** (`/etc/sssd/sssd.conf`)

5. **Настройка PAM** для аутентификации через AD

## Пример реализации роли

```yaml
---
# файл: roles/ad-integration/tasks/main.yml

- name: Install required packages
  apt:
    name: "{{ item }}"
    state: present
    update_cache: yes
  loop:
    - realmd
    - sssd
    - sssd-tools
    - libnss-sss
    - libpam-sss
    - adcli
    - samba-common-bin
    - oddjob
    - oddjob-mkhomedir
    - krb5-user
  when: ansible_os_family == 'Debian'

- name: Configure Kerberos
  template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
    owner: root
    group: root
    mode: 0644

- name: Join Active Directory domain
  command: |
    realm join {{ ad_domain }} \
      --user="{{ ad_admin_username }}" \
      --computer-ou="{{ ad_computer_ou }}" \
      --verbose
  args:
    stdin: "{{ ad_admin_password }}"
  register: realm_join_result
  failed_when: "'Already joined to this domain' not in realm_join_result.stderr and realm_join_result.rc != 0"
  changed_when: "'Already joined to this domain' not in realm_join_result.stderr"

- name: Configure SSSD
  template:
    src: sssd.conf.j2
    dest: /etc/sssd/sssd.conf
    owner: root
    group: root
    mode: 0600
  notify:
    - restart sssd

- name: Enable automatic home directory creation
  lineinfile:
    path: /etc/pam.d/common-session
    line: "session required pam_mkhomedir.so skel=/etc/skel/ umask=0022"
    state: present

- name: Configure sudo for AD groups
  copy:
    src: sudoers_ad
    dest: /etc/sudoers.d/sudoers_ad
    mode: 0440
...
```

## Шаблоны конфигурационных файлов

`krb5.conf.j2`:
```ini
[libdefaults]
    default_realm = {{ ad_realm | upper }}
    dns_lookup_realm = true
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false

[realms]
    {{ ad_realm | upper }} = {
        kdc = {{ ad_domain_controller }}
        admin_server = {{ ad_domain_controller }}
    }

[domain_realm]
    .{{ ad_domain }} = {{ ad_realm | upper }}
    {{ ad_domain }} = {{ ad_realm | upper }}
```

`sssd.conf.j2`:
```ini
[sssd]
domains = {{ ad_domain }}
config_file_version = 2
services = nss, pam

[domain/{{ ad_domain }}]
ad_domain = {{ ad_domain }}
krb5_realm = {{ ad_realm | upper }}
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
id_provider = ad
krb5_store_password_if_offline = True
default_shell = /bin/bash
ldap_id_mapping = True
use_fully_qualified_names = False
fallback_homedir = /home/%u
access_provider = ad
```

## Переменные роли

Создайте файл `roles/ad-integration/defaults/main.yml`:
```yaml
---
ad_domain: example.com
ad_realm: EXAMPLE.COM
ad_domain_controller: dc1.example.com
ad_admin_username: "adminuser"
ad_admin_password: "password"
ad_computer_ou: "OU=Linux,OU=Computers,DC=example,DC=com"
```

## Обработчики

Добавьте в `roles/ad-integration/handlers/main.yml`:
```yaml
---
- name: restart sssd
  service:
    name: sssd
    state: restarted
```

## Использование роли

1. Создайте playbook (`ad-join.yml`):
```yaml
---
- hosts: linux_servers
  become: yes
  roles:
    - ad-integration
```

2. Запустите playbook:
```bash
ansible-playbook -i inventory ad-join.yml --extra-vars "ad_admin_password=YourADAdminPassword"
```

## Дополнительные рекомендации

1. Используйте Ansible Vault для безопасного хранения паролей
2. Настройте ограничения sudo для AD-групп
3. Проверьте правильность настроек DNS на Linux-сервере
4. Убедитесь, что сервер может разрешать имена контроллеров домена
5. Рассмотрите возможность настройки автоматического обновления Kerberos-тикетов

Для более сложных сценариев можно добавить:
- Настройку автоматического монтирования домашних каталогов
- Интеграцию с Samba для общего доступа к файлам
- Настройку SSH-доступа для AD-пользователей