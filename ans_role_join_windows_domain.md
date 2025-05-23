# Ansible роль для ввода Oracle Linux 9 в домен Windows

Ниже представлена базовая Ansible роль для присоединения Oracle Linux 9 к домену Windows с использованием realmd и SSSD.

## Структура роли

```
roles/join_windows_domain/
├── tasks/
│   └── main.yml
├── vars/
│   └── main.yml
├── templates/
│   └── sssd.conf.j2
└── README.md
```

## Файлы роли

### vars/main.yml

```yaml
# Учетные данные домена
domain_name: "example.com"
domain_admin_user: "joinuser"
domain_admin_password: "securepassword"

# Настройки OU (опционально)
computer_ou: "OU=Linux Servers,DC=example,DC=com"

# Настройки SSSD
sssd_config:
  domains: "{{ domain_name }}"
  config_file: "/etc/sssd/sssd.conf"
  services: "nss, pam"
```

### tasks/main.yml

```yaml
---
- name: Установка необходимых пакетов
  dnf:
    name:
      - realm
      - sssd
      - sssd-tools
      - sssd-ad
      - adcli
      - samba-common-tools
      - krb5-workstation
      - oddjob
      - oddjob-mkhomedir
    state: present
    update_cache: yes

- name: Настройка Kerberos (если требуется)
  template:
    src: krb5.conf.j2
    dest: /etc/krb5.conf
    owner: root
    group: root
    mode: 0644
  when: configure_krb5 | default(false)

- name: Присоединение к домену
  command: |
    echo "{{ domain_admin_password }}" | realm join \
    --user="{{ domain_admin_user }}" \
    {% if computer_ou is defined %}--computer-ou="{{ computer_ou }}" {% endif %}
    --verbose {{ domain_name }}
  args:
    creates: "/etc/sssd/sssd.conf"
  register: realm_join_result
  changed_when: "'already joined' not in realm_join_result.stderr"
  no_log: true

- name: Настройка SSSD
  template:
    src: sssd.conf.j2
    dest: "{{ sssd_config.config_file }}"
    owner: root
    group: root
    mode: 0600
  notify:
    - Restart SSSD service

- name: Включение и запуск SSSD
  service:
    name: sssd
    state: started
    enabled: yes

- name: Настройка автоматического создания домашних директорий
  service:
    name: oddjobd
    state: started
    enabled: yes

- name: Настройка прав доступа к sudoers из AD (опционально)
  lineinfile:
    path: /etc/nsswitch.conf
    regexp: '^sudoers:'
    line: 'sudoers: files sss'
    state: present
```

### templates/sssd.conf.j2

```jinja2
[sssd]
domains = {{ sssd_config.domains }}
services = {{ sssd_config.services }}
config_file_version = 2

[domain/{{ domain_name }}]
id_provider = ad
access_provider = ad
ad_domain = {{ domain_name }}
krb5_realm = {{ domain_name | upper }}
realmd_tags = manages-system joined-with-adcli
cache_credentials = True
krb5_store_password_if_offline = True
default_shell = /bin/bash
ldap_id_mapping = True
use_fully_qualified_names = False
fallback_homedir = /home/%u
```

## Использование роли

1. Создайте playbook (например, `join_domain.yml`):

```yaml
---
- hosts: linux_servers
  become: yes
  roles:
    - join_windows_domain
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory join_domain.yml --extra-vars "domain_admin_password=secretpass"
```

## Дополнительные настройки

1. **Sudoers из AD**: Для управления правами sudo через AD, создайте группу в AD и добавьте в `/etc/sudoers` строку:
   ```
   %DOMAIN\ LinuxAdmins ALL=(ALL) ALL
   ```

2. **Устранение неполадок**:
   - Проверьте журналы: `journalctl -xe`
   - Проверьте подключение к домену: `realm list`
   - Проверьте аутентификацию: `id user@domain`

3. **Безопасность**:
   - Используйте Ansible Vault для хранения паролей
   - Ограничьте права учетной записи для присоединения к домену

Эта роль предоставляет базовую функциональность. В зависимости от требований вашего окружения могут потребоваться дополнительные настройки.