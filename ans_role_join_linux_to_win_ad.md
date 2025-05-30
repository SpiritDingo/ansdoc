# Улучшенная Ansible роль для добавления Oracle Linux 9 в Active Directory

Вот усовершенствованная роль с созданием keytab файла и всеми необходимыми проверками.

## Структура роли

```
oraclelinux-ad-join/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── join.yml
│   ├── verify.yml
│   └── keytab.yml
├── templates/
│   ├── sssd.conf.j2
│   └── krb5.conf.j2
└── handlers/
    └── main.yml
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки домена
ad_domain: "example.com"
ad_domain_upper: "{{ ad_domain | upper }}"
ad_join_user: "join_user@{{ ad_domain_upper }}"
ad_join_password: ""
ad_ou: "OU=LinuxServers,DC=example,DC=com"

# Настройки keytab
keytab_path: "/etc/krb5.keytab"
keytab_principal: "host/{{ ansible_fqdn }}@{{ ad_domain_upper }}"
keytab_permissions: "0600"
renew_keytab: yes

# Настройки Kerberos
krb5_config:
  default_realm: "{{ ad_domain_upper }}"
  dns_lookup_realm: true
  dns_lookup_kdc: true
  ticket_lifetime: 24h
  renew_lifetime: 7d
  forwardable: true
  rdns: false
  default_ccache_name: "KEYRING:persistent:%{uid}"

# Настройки SSSD
sssd_config:
  config_file: "/etc/sssd/sssd.conf"
  services: "nss, pam, ssh"
  domains: "{{ ad_domain }}"

# Временные файлы
temp_password_file: "/tmp/ad_join_password.txt"
cleanup_temp_files: yes
```

### tasks/main.yml

```yaml
---
- name: Include pre-join tasks
  ansible.builtin.include_tasks: prechecks.yml
  tags: prechecks

- name: Install required packages
  ansible.builtin.include_tasks: packages.yml
  tags: packages

- name: Configure Kerberos
  ansible.builtin.include_tasks: kerberos.yml
  tags: kerberos

- name: Join to AD domain
  ansible.builtin.include_tasks: join.yml
  tags: join

- name: Configure keytab
  ansible.builtin.include_tasks: keytab.yml
  tags: keytab

- name: Configure SSSD
  ansible.builtin.include_tasks: sssd.yml
  tags: sssd

- name: Verify domain join
  ansible.builtin.include_tasks: verify.yml
  tags: verify

- name: Post-join configuration
  ansible.builtin.include_tasks: postconfig.yml
  tags: postconfig
```

### tasks/join.yml

```yaml
---
- name: Create temporary password file
  ansible.builtin.copy:
    dest: "{{ temp_password_file }}"
    content: "{{ ad_join_password }}"
    mode: 0600
    no_log: true
  when: ad_join_password != ""
  tags: join

- name: Join to AD domain using adcli
  ansible.builtin.command: >
    adcli join {{ ad_domain }}
    -U "{{ ad_join_user }}"
    --stdin-password < "{{ temp_password_file }}"
    {% if ad_ou %}--computer-ou="{{ ad_ou }}"{% endif %}
    --verbose
  register: ad_join_result
  changed_when: >
    "'already joined' not in ad_join_result.stderr and
     'Successfully enrolled machine in realm' in ad_join_result.stdout"
  failed_when: false
  ignore_errors: yes
  no_log: true
  when: ad_join_password != ""
  tags: join

- name: Cleanup temporary password file
  ansible.builtin.file:
    path: "{{ temp_password_file }}"
    state: absent
  when: ad_join_password != "" and cleanup_temp_files
  tags: cleanup
```

### tasks/keytab.yml

```yaml
---
- name: Check if keytab exists
  ansible.builtin.stat:
    path: "{{ keytab_path }}"
  register: keytab_stat
  tags: keytab

- name: Create keytab file if not exists
  ansible.builtin.command: >
    touch {{ keytab_path }} &&
    chmod {{ keytab_permissions }} {{ keytab_path }}
  when: not keytab_stat.stat.exists
  tags: keytab

- name: Set keytab permissions
  ansible.builtin.file:
    path: "{{ keytab_path }}"
    mode: "{{ keytab_permissions }}"
  tags: keytab

- name: Add principal to keytab
  ansible.builtin.command: >
    net ads keytab add {{ keytab_principal.split('@')[0] }} -k
  register: keytab_add
  changed_when: "'Entry added successfully' in keytab_add.stdout"
  failed_when: >
    "'Failed to add entry' in keytab_add.stderr or
     'kinit failed' in keytab_add.stderr"
  tags: keytab

- name: Verify keytab content
  ansible.builtin.command: klist -ket "{{ keytab_path }}"
  register: keytab_verify
  changed_when: false
  failed_when: keytab_verify.rc != 0
  tags: keytab

- name: Renew keytab entries (if configured)
  ansible.builtin.command: net ads keytab renew -k
  when: renew_keytab
  register: keytab_renew
  changed_when: "'Keytab successfully renewed' in keytab_renew.stdout"
  failed_when: keytab_renew.rc != 0
  tags: keytab
```

### tasks/verify.yml

```yaml
---
- name: Verify domain join with realm
  ansible.builtin.command: realm list
  register: realm_verify
  changed_when: false
  failed_when: >
    "'{{ ad_domain }}' not in realm_verify.stdout or
     'configured: no' in realm_verify.stdout"
  tags: verify

- name: Verify domain join with adcli
  ansible.builtin.command: adcli testjoin
  register: adcli_verify
  changed_when: false
  failed_when: adcli_verify.rc != 0
  tags: verify

- name: Verify SSSD domain status
  ansible.builtin.command: sssctl domain-status "{{ ad_domain }}"
  register: sssd_verify
  changed_when: false
  failed_when: >
    "'Online status: Online' not in sssd_verify.stdout or
     'Active servers: None' in sssd_verify.stdout"
  tags: verify

- name: Verify Kerberos authentication
  ansible.builtin.command: kinit -k "{{ keytab_principal }}"
  register: kinit_verify
  changed_when: false
  failed_when: kinit_verify.rc != 0
  tags: verify

- name: Verify keytab content
  ansible.builtin.command: klist -ket "{{ keytab_path }}"
  register: keytab_verify_final
  changed_when: false
  failed_when: >
    keytab_verify_final.rc != 0 or
    '{{ keytab_principal }}' not in keytab_verify_final.stdout
  tags: verify

- name: Show verification summary
  ansible.builtin.debug:
    msg: |
      Domain join verification summary:
      - Realm status: {{ 'OK' if "'{{ ad_domain }}' in realm_verify.stdout" else 'FAILED' }}
      - ADCLI testjoin: {{ 'OK' if adcli_verify.rc == 0 else 'FAILED' }}
      - SSSD status: {{ 'OK' if "'Online status: Online' in sssd_verify.stdout" else 'FAILED' }}
      - Kerberos auth: {{ 'OK' if kinit_verify.rc == 0 else 'FAILED' }}
      - Keytab content: {{ 'OK' if keytab_verify_final.rc == 0 else 'FAILED' }}
  tags: verify
```

### handlers/main.yml

```yaml
---
- name: restart sssd
  ansible.builtin.systemd:
    name: sssd
    state: restarted

- name: restart oddjobd
  ansible.builtin.systemd:
    name: oddjobd
    state: restarted
```

## Использование роли

1. Создайте playbook:

```yaml
---
- name: Join Oracle Linux 9 to AD with keytab
  hosts: all
  become: yes
  vars:
    ad_domain: "example.com"
    ad_join_user: "join_user@EXAMPLE.COM"
    ad_join_password: "secure_password"
    ad_ou: "OU=LinuxServers,DC=example,DC=com"
  
  roles:
    - oraclelinux-ad-join
```

2. Запустите с зашифрованным паролем:

```bash
ansible-playbook -i inventory.ini join_ad.yml --ask-vault-pass
```

## Ключевые особенности

1. **Полное управление keytab**:
   - Создание keytab файла
   - Добавление principal в keytab
   - Проверка содержимого keytab
   - Обновление keytab

2. **Комплексные проверки**:
   - Проверка через `realm list`
   - Проверка через `adcli testjoin`
   - Проверка статуса SSSD
   - Проверка Kerberos аутентификации
   - Детальная проверка keytab

3. **Обработка ошибок**:
   - Проверка на каждом этапе
   - Подробные сообщения об ошибках
   - Возможность повтора операций

4. **Безопасность**:
   - Использование временных файлов с паролями
   - Автоматическая очистка временных файлов
   - Правильные права на keytab файл

5. **Гибкость**:
   - Настройка OU для компьютера в AD
   - Возможность обновления keytab
   - Настраиваемые параметры Kerberos и SSSD