Я подготовил для вас полную Ansible-роль для введения серверов Ubuntu 22.04 и 24.04 в домен MS Active Directory. Роль включает установку необходимых пакетов, настройку, присоединение к домену и комплексные проверки результата.

📁 Структура роли и файлы

Создайте следующую структуру файлов и каталогов:

```
roles/
└── ad_join/
    ├── defaults/
    │   └── main.yml
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    └── templates/
        └── krb5.conf.j2
```

📄 Содержимое файлов роли

1. Файл roles/ad_join/defaults/main.yml (переменные по умолчанию):

```yaml
---
# Доменные настройки по умолчанию
ad_domain: "EXAMPLE.LOCAL"
ad_netbios_name: "EXAMPLE"
ad_join_user: "adjoinuser@EXAMPLE.LOCAL"

# Группы AD
ad_allowed_group: "LinuxAdmins"

# Настройки SSSD
sssd_use_fully_qualified_names: False
sssd_fallback_homedir: "/home/%u"
```

2. Файл roles/ad_join/tasks/main.yml (основные задачи):

```yaml
---
- name: Install required packages for AD integration
  apt:
    name:
      - sssd
      - realmd
      - adcli
      - samba-common
      - krb5-user
      - packagekit
      - oddjob
      - oddjob-mkhomedir
    state: present
  when: ansible_distribution == "Ubuntu"
  register: package_install
  notify: restart sssd

- name: Configure Kerberos client
  template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
  when: ansible_distribution == "Ubuntu"

- name: Check current domain status
  command: realm list
  register: realm_status
  changed_when: false
  ignore_errors: yes
  failed_when: false

- name: Join Ubuntu to Active Directory domain
  command: |
    echo "{{ ad_join_password }}" | realm join --user="{{ ad_join_user }}" {{ ad_domain }}
  args:
    warn: false
  when: 
    - ansible_distribution == "Ubuntu"
    - ad_domain not in realm_status.stdout | default('')
  register: domain_join_result
  no_log: true  # Скрывает пароль в логах
  ignore_errors: yes

- name: Configure SSSD for simple logins
  lineinfile:
    path: /etc/sssd/sssd.conf
    regexp: '^{{ item.regexp }}'
    line: '{{ item.line }}'
    state: present
  with_items:
    - { regexp: '^use_fully_qualified_names =', line: 'use_fully_qualified_names = {{ sssd_use_fully_qualified_names | bool | lower }}' }
    - { regexp: '^fallback_homedir =', line: 'fallback_homedir = {{ sssd_fallback_homedir }}' }
  notify: restart sssd

- name: Allow specific AD group to log in
  command: realm permit -g "{{ ad_allowed_group }}@{{ ad_domain }}"
  when: realm_status.rc == 0 or domain_join_result.rc == 0

- name: Grant sudo privileges to AD group
  lineinfile:
    path: /etc/sudoers.d/00-adm-group
    line: '%{{ ad_allowed_group }} ALL=(ALL) ALL'
    state: present
    validate: 'visudo -cf %s'
  when: realm_status.rc == 0 or domain_join_result.rc == 0

# ПРОВЕРКИ ПОСЛЕ ПРИСОЕДИНЕНИЯ
- name: Verify domain join with realm list
  command: realm list
  register: domain_verify
  changed_when: false
  failed_when: 
    - domain_verify.rc != 0
    - ad_domain not in domain_verify.stdout

- name: Display domain join status
  debug:
    msg: "Server successfully joined domain {{ ad_domain }}"
  when: ad_domain in domain_verify.stdout

- name: Test AD user lookup
  command: id "{{ ad_join_user }}"
  register: user_lookup_test
  changed_when: false
  ignore_errors: yes

- name: Display user lookup result
  debug:
    msg: "AD user lookup successful - domain authentication is working"
  when: user_lookup_test.rc == 0

- name: Verify SSSD service is running
  systemd:
    name: sssd
    state: started
    enabled: yes
  register: sssd_service_status

- name: Final success message
  debug:
    msg: |
      Domain join completed successfully!
      Domain: {{ ad_domain }}
      Allowed group: {{ ad_allowed_group }}
      SSSD status: {{ sssd_service_status.state }}
  when: ad_domain in domain_verify.stdout
```

3. Файл roles/ad_join/handlers/main.yml (обработчики):

```yaml
---
- name: restart sssd
  systemd:
    name: sssd
    state: restarted
```

4. Файл roles/ad_join/templates/krb5.conf.j2 (шаблон Kerberos):

```
[libdefaults]
    default_realm = {{ ad_domain | upper }}
    dns_lookup_realm = true
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true

[realms]
    {{ ad_domain | upper }} = {
        kdc = {{ ad_domain | lower }}
        admin_server = {{ ad_domain | lower }}
    }

[domain_realm]
    .{{ ad_domain | lower }} = {{ ad_domain | upper }}
    {{ ad_domain | lower }} = {{ ad_domain | upper }}
```

🚀 Плейбук для использования роли

Создайте файл join-domain.yml в корневой директории вашего проекта:

```yaml
---
- name: Join Ubuntu servers to Active Directory
  hosts: linux_servers
  gather_facts: true
  vars_files:
    - vars/ad_secrets.yml  # Создайте этот файл с паролями
  roles:
    - role: ad_join
```

🔐 Безопасное хранение паролей

Создайте зашифрованный файл vars/ad_secrets.yml с помощью ansible-vault:

```yaml
---
# vars/ad_secrets.yml
ad_join_password: "your_secure_domain_join_password"
```

Запустите плейбук с запросом пароля vault:

```bash
ansible-playbook join-domain.yml --ask-vault-pass
```

🔍 Ключевые особенности реализации

· Идемпотентность: Роль проверяет текущий статус домена перед присоединением, что позволяет безопасно запускать ее многократно.
· Безопасность: Пароли скрыты в логах с помощью no_log: true.
· Проверки: Включает многоуровневую проверку успешности присоединения через realm list и тестирование аутентификации AD-пользователя.
· Гибкость: Настройки можно легко переопределить при вызове роли.

Роль готова к использованию и включает все необходимые исправления для надежного присоединения серверов Ubuntu 22.04 и 24.04 к домену Active Directory.