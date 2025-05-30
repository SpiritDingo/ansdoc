# Роль Ansible для добавления Oracle Linux 9 в Active Directory

Вот пример роли Ansible для присоединения Oracle Linux 9 к домену Active Directory с проверками keytab, обработкой ошибок и проверкой успешного присоединения.

## Структура роли

```
oraclelinux-ad-join/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
├── templates/
│   └── sssd.conf.j2
└── meta/
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки домена
ad_domain: "example.com"
ad_join_user: "join_user@EXAMPLE.COM"
ad_join_password: ""

# Настройки SSSD
sssd_config:
  domains: "{{ ad_domain }}"
  config_file: "/etc/sssd/sssd.conf"
  services: "nss, pam, ssh"

# Настройки Kerberos
krb5_config:
  default_realm: "{{ ad_domain | upper }}"
  dns_lookup_realm: true
  dns_lookup_kdc: true
  ticket_lifetime: 24h
  renew_lifetime: 7d
  forwardable: true
  rdns: false
  default_ccache_name: "KEYRING:persistent:%{uid}"

# Путь к keytab файлу
keytab_path: "/etc/krb5.keytab"

# Временный файл для пароля
temp_password_file: "/tmp/ad_join_password.txt"
```

### tasks/main.yml

```yaml
---
- name: Установка необходимых пакетов
  dnf:
    name:
      - sssd
      - sssd-ad
      - sssd-tools
      - adcli
      - samba-common-tools
      - krb5-workstation
      - oddjob
      - oddjob-mkhomedir
      - authselect
    state: present
  tags: packages

- name: Настройка Kerberos
  template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
    owner: root
    group: root
    mode: 0644
  tags: kerberos

- name: Создание временного файла с паролем
  copy:
    dest: "{{ temp_password_file }}"
    content: "{{ ad_join_password }}"
    mode: 0600
  when: ad_join_password != ""
  tags: join

- name: Присоединение к домену AD
  command: >
    adcli join {{ ad_domain }}
    -U "{{ ad_join_user }}"
    --stdin-password < "{{ temp_password_file }}"
    --verbose
  register: ad_join_result
  changed_when: "'already joined' not in ad_join_result.stderr"
  failed_when: false
  ignore_errors: yes
  when: ad_join_password != ""
  tags: join

- name: Проверка успешности присоединения к домену
  block:
    - name: Проверка существования keytab файла
      stat:
        path: "{{ keytab_path }}"
      register: keytab_file

    - name: Проверка содержимого keytab
      command: klist -ke "{{ keytab_path }}"
      register: keytab_check
      when: keytab_file.stat.exists

    - name: Проверка членства в домене с помощью adcli
      command: adcli info "{{ ad_domain }}"
      register: domain_info

    - name: Проверка через realm
      command: realm list
      register: realm_info

    - name: Проверка через sssctl
      command: sssctl domain-status "{{ ad_domain }}"
      register: sssctl_info

    - name: Завершение с ошибкой если присоединение не удалось
      fail:
        msg: |
          Присоединение к домену не удалось:
          - keytab exists: {{ keytab_file.stat.exists }}
          - keytab valid: {{ keytab_check is success }}
          - adcli info: {{ domain_info.rc == 0 }}
          - realm list: {{ 'online' in realm_info.stdout }}
          - sssctl status: {{ 'online' in sssctl_info.stdout }}
      when: >
        not keytab_file.stat.exists or
        (keytab_file.stat.exists and keytab_check is failed) or
        domain_info.rc != 0 or
        'online' not in realm_info.stdout or
        'online' not in sssctl_info.stdout
  tags: verify

- name: Настройка SSSD
  template:
    src: sssd.conf.j2
    dest: "{{ sssd_config.config_file }}"
    owner: root
    group: root
    mode: 0600
  notify: restart sssd
  tags: sssd

- name: Настройка authselect
  command: authselect select sssd with-mkhomedir --force
  tags: auth

- name: Включение oddjobd для создания домашних директорий
  systemd:
    name: oddjobd
    state: started
    enabled: yes
  tags: oddjob

- name: Очистка временного файла с паролем
  file:
    path: "{{ temp_password_file }}"
    state: absent
  when: ad_join_password != ""
  tags: cleanup

- name: Перезапуск SSSD
  handlers:
    - name: restart sssd
      systemd:
        name: sssd
        state: restarted
      tags: restart
```

### templates/sssd.conf.j2

```jinja2
[sssd]
domains = {{ sssd_config.domains }}
services = {{ sssd_config.services }}
config_file_version = 2

[domain/{{ ad_domain }}]
id_provider = ad
access_provider = ad
auth_provider = ad
chpass_provider = ad
ad_domain = {{ ad_domain }}
ad_server = {{ ad_domain }}
ad_hostname = {{ ansible_hostname }}.{{ ad_domain }}
krb5_realm = {{ ad_domain | upper }}
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
enumerate = False
ldap_id_mapping = True
fallback_homedir = /home/%u
default_shell = /bin/bash
```

### templates/krb5.conf.j2

```jinja2
[libdefaults]
    default_realm = {{ krb5_config.default_realm }}
    dns_lookup_realm = {{ krb5_config.dns_lookup_realm | lower }}
    dns_lookup_kdc = {{ krb5_config.dns_lookup_kdc | lower }}
    ticket_lifetime = {{ krb5_config.ticket_lifetime }}
    renew_lifetime = {{ krb5_config.renew_lifetime }}
    forwardable = {{ krb5_config.forwardable | lower }}
    rdns = {{ krb5_config.rdns | lower }}
    default_ccache_name = {{ krb5_config.default_ccache_name }}

[realms]
    {{ krb5_config.default_realm }} = {
    }

[domain_realm]
    .{{ ad_domain }} = {{ krb5_config.default_realm }}
    {{ ad_domain }} = {{ krb5_config.default_realm }}
```

## Использование роли

1. Создайте playbook для использования роли:

```yaml
---
- name: Join Oracle Linux 9 to AD
  hosts: all
  become: yes
  vars:
    ad_domain: "example.com"
    ad_join_user: "join_user@EXAMPLE.COM"
    ad_join_password: "secure_password"
  
  roles:
    - oraclelinux-ad-join
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini join_ad.yml
```

## Проверки в роли

Роль включает следующие проверки:
1. Проверка существования keytab файла (`/etc/krb5.keytab`)
2. Проверка валидности keytab с помощью `klist -ke`
3. Проверка членства в домене через `adcli info`
4. Проверка статуса через `realm list`
5. Проверка статуса домена через `sssctl domain-status`

Если какая-либо из проверок не проходит, роль завершается с ошибкой и подробным сообщением.

## Дополнительные улучшения

1. Добавьте обработку различных ошибок AD (неправильные учетные данные, проблемы с DNS и т.д.)
2. Реализуйте поддержку нескольких доменов
3. Добавьте настройку firewall для работы с AD
4. Реализуйте проверку времени и синхронизацию NTP перед присоединением к домену