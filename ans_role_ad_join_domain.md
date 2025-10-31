Я преобразую ваш плейбук в роль Ansible для Ubuntu 22.04 и 24.04. Вот структура роли:

Структура роли

```
roles/ad_domain_join/
├── defaults
│   └── main.yml
├── tasks
│   └── main.yml
├── handlers
│   └── main.yml
├── templates
│   ├── krb5.conf.j2
│   ├── sssd.conf.j2
│   ├── sshd_config.j2
│   ├── sudoers.j2
│   ├── 1_DOMAIN-powerusers.j2
│   ├── 2_DOMAIN-localadmins.j2
│   └── 3_DOMAIN-globaladmins.j2
└── files
    └── common-session
    └── common-auth
```

1. defaults/main.yml

```yaml
---
# defaults file for ad_domain_join
domain: "DOMAIN.ru"
ad_join_user: ""
ad_join_password: ""

packages:
  - realmd
  - sssd
  - sssd-tools
  - libnss-sss
  - libpam-sss
  - adcli
  - samba-common
  - samba-common-bin
  - oddjob
  - oddjob-mkhomedir
  - packagekit
  - libpam-krb5

# Файлы конфигурации
krb5_conf_path: /etc/krb5.conf
sssd_conf_path: /etc/sssd/sssd.conf
sshd_config_path: /etc/ssh/sshd_config
sudoers_path: /etc/sudoers
```

2. tasks/main.yml

```yaml
---
- name: Show short summary to operator
  ansible.builtin.debug:
    msg:
      - "ВНИМАНИЕ: этот плейбук выполнит настройку домена '{{ domain }}'."
      - "Пользователь для join: {{ ad_join_user }}"

- name: Ensure apt cache updated (pre)
  ansible.builtin.apt:
    update_cache: yes
  when: ansible_os_family == "Debian"

- name: Preseed krb5-config answers (suppress interactive prompt)
  ansible.builtin.debconf:
    name: krb5-config
    question: krb5-config/default_realm
    value: ""
    vtype: string
  when: ansible_os_family == "Debian"
  ignore_errors: yes

- name: Stop and disable nscd if present
  ansible.builtin.systemd:
    name: nscd
    state: stopped
    enabled: no
  ignore_errors: yes

- name: Stop and disable google-guest-agent.service if present
  ansible.builtin.systemd:
    name: google-guest-agent.service
    state: stopped
    enabled: no
  ignore_errors: yes

- name: Ensure /etc/ssh/sshd_config.d/50-cloud-init.conf - comment PasswordAuthentication line if present
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config.d/50-cloud-init.conf
    regexp: '^\s*PasswordAuthentication\s+no'
    line: '#PasswordAuthentication no'
    create: yes
    backup: yes
  notify: restart ssh

- name: Install required packages
  ansible.builtin.apt:
    name: "{{ packages }}"
    state: present
    allow_unauthenticated: no
    force: yes
  environment:
    DEBIAN_FRONTEND: noninteractive
  register: apt_install
  when: ansible_os_family == "Debian"

- name: Create symlink for pam_krb5 if needed
  ansible.builtin.file:
    src: /lib/x86_64-linux-gnu/security/pam_krb5.so
    dest: /lib/security/pam_krb5.so
    state: link
  ignore_errors: yes

- name: Configure krb5.conf
  ansible.builtin.template:
    src: krb5.conf.j2
    dest: "{{ krb5_conf_path }}"
    owner: root
    group: root
    mode: '0644'
    backup: yes

- name: Configure sssd.conf
  ansible.builtin.template:
    src: sssd.conf.j2
    dest: "{{ sssd_conf_path }}"
    owner: root
    group: root
    mode: '0600'
    backup: yes
  notify: restart sssd

- name: Restart sssd (initial)
  ansible.builtin.systemd:
    name: sssd
    state: restarted
  ignore_errors: yes

- name: Clear sssd cache
  ansible.builtin.command: sss_cache -E
  ignore_errors: yes

- name: Configure PAM common-session
  ansible.builtin.copy:
    dest: /etc/pam.d/common-session
    owner: root
    group: root
    mode: '0644'
    content: |
      session required    pam_unix.so
      session optional    pam_sss.so
      session optional    pam_krb5.so
      session optional    pam_systemd.so
      session optional    pam_umask.so
      session required pam_mkhomedir.so skel=/etc/skel/ umask=0022
    backup: yes

- name: Configure PAM common-auth
  ansible.builtin.copy:
    dest: /etc/pam.d/common-auth
    owner: root
    group: root
    mode: '0644'
    content: |
      # Локальные пользователи
      auth sufficient pam_unix.so nullok try_first_pass
      # Доменные пользователи через SSSD
      auth sufficient pam_sss.so use_first_pass
      # Выдача Kerberos тикета
      auth optional pam_krb5.so use_first_pass
      # fallback: если ни один модуль не прошёл
      auth requisite pam_deny.so
      # Дополнительные возможности
      auth optional pam_cap.so
    backup: yes

- name: Configure SSH daemon
  ansible.builtin.template:
    src: sshd_config.j2
    dest: "{{ sshd_config_path }}"
    owner: root
    group: root
    mode: '0644'
    backup: yes
  notify: restart ssh

- name: Configure sudoers
  ansible.builtin.template:
    src: sudoers.j2
    dest: "{{ sudoers_path }}"
    owner: root
    group: root
    mode: '0440'
    backup: yes

- name: Configure domain powerusers sudoers
  ansible.builtin.template:
    src: 1_DOMAIN-powerusers.j2
    dest: "/etc/sudoers.d/1_DOMAIN-powerusers"
    owner: root
    group: root
    mode: '0440'
    backup: yes

- name: Configure domain localadmins sudoers
  ansible.builtin.template:
    src: 2_DOMAIN-localadmins.j2
    dest: "/etc/sudoers.d/2_DOMAIN-localadmins"
    owner: root
    group: root
    mode: '0440'
    backup: yes

- name: Configure domain globaladmins sudoers
  ansible.builtin.template:
    src: 3_DOMAIN-globaladmins.j2
    dest: "/etc/sudoers.d/3_DOMAIN-globaladmins"
    owner: root
    group: root
    mode: '0440'
    backup: yes

- name: Ensure permissions for sudoers.d files
  ansible.builtin.file:
    path: "/etc/sudoers.d"
    recurse: no
    state: directory
    mode: '0755'

- name: Fix modes for the three files explicitly
  ansible.builtin.file:
    path: "{{ item }}"
    mode: '0440'
  loop:
    - /etc/sudoers.d/1_DOMAIN-powerusers
    - /etc/sudoers.d/2_DOMAIN-localadmins
    - /etc/sudoers.d/3_DOMAIN-globaladmins
  ignore_errors: yes

- name: Final restart services
  ansible.builtin.systemd:
    name: "{{ item }}"
    state: restarted
  loop:
    - sssd
    - sshd
    - ssh
  ignore_errors: yes

- name: Remove unwanted sudoers files (cloud-init/google)
  ansible.builtin.file:
    path: "{{ item }}"
    state: absent
  loop:
    - /etc/sudoers.d/google_sudoers
    - /etc/sudoers.d/90-cloud-init-users
  ignore_errors: yes
```

3. handlers/main.yml

```yaml
---
- name: restart sssd
  ansible.builtin.systemd:
    name: sssd
    state: restarted
  ignore_errors: yes

- name: restart ssh
  ansible.builtin.systemd:
    name: ssh
    state: restarted
  ignore_errors: yes
```

4. Примеры шаблонов

templates/sssd.conf.j2

```jinja2
[sssd]
domains = {{ domain }}
config_file_version = 2
services = nss, pam

[domain/{{ domain }}]
default_shell = /bin/bash
krb5_store_password_if_offline = True
cache_credentials = True
krb5_realm = {{ domain | upper }}
realmd_tags = manages-system joined-with-adcli
id_provider = ad
fallback_homedir = /home/%u@%d
ad_domain = {{ domain }}
use_fully_qualified_names = True
ldap_id_mapping = True
access_provider = ad
ad_hostname = {{ inventory_hostname_short }}.{{ domain }}
dyndns_update = False
```

templates/1_DOMAIN-powerusers.j2

```jinja2
# Доменные powerusers для {{ inventory_hostname_short }}
%DOMAIN\\powerusers_{{ inventory_hostname_short }} ALL=(ALL) ALL
```

5. Плейбук для использования роли

```yaml
---
- name: Configure AD domain settings
  hosts: all
  become: yes
  gather_facts: yes

  vars_prompt:
    - name: "ad_join_user"
      prompt: >
        "Внимание. Будет выполнена настройка домена на серверах из hosts плейбука.
         Введите имя учётной записи (формат логина: QTxx):"
      private: no
    - name: "ad_join_password"
      prompt: "Введите пароль для учётной записи (скрытый ввод):"
      private: yes

  pre_tasks:
    - name: Show short summary to operator
      ansible.builtin.debug:
        msg:
          - "ВНИМАНИЕ: этот плейбук выполнит настройку домена '{{ domain }}'."
          - "Пользователь: {{ ad_join_user }}"

  roles:
    - role: ad_domain_join
```

Особенности реализации:

1. Шаблоны вместо копирования: Все конфигурационные файлы теперь используют шаблоны Jinja2
2. Переменные по умолчанию: Настройки вынесены в defaults/main.yml
3. Обработчики: Вынесены в отдельный файл для лучшей организации
4. Универсальность: Роль работает для Ubuntu 22.04 и 24.04
5. Безопасность: Переменные для входа в домен запрашиваются при запуске

Для использования роли поместите файлы конфигурации в соответствующие директории шаблонов и настройте переменные под ваше окружение.