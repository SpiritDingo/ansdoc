Ниже представлена полная Ansible-роль os_repo, собранная на основе обсуждения в файле. Роль поддерживает Ubuntu (Jammy/Noble) и Oracle Linux (8/9), позволяет настраивать несколько репозиториев, скачивать GPG-ключи с локального репозитория (в том числе с Basic Auth), использовать DEB822 и .repo форматы, а также управлять правами доступа к файлам с учётными данными.

Структура роли

```
roles/os_repo/
├── defaults/
│   └── main.yml
├── handlers/
│   └── main.yml
├── tasks/
│   ├── main.yml
│   ├── ubuntu.yml
│   └── oraclelinux.yml
└── templates/
    ├── ubuntu.sources.j2
    ├── ubuntu.auth.conf.j2
    └── oraclelinux.repo.j2
```

Содержимое файлов

defaults/main.yml

```yaml
---
# ===========================================
# Настройки для Ubuntu (Jammy / Noble)
# ===========================================
ubuntu_repositories:
  - filename: "local-ubuntu.sources"
    types: "deb"
    uris: "https://mirror.local"
    suites:
      - "{{ ansible_distribution_release }}"
      - "{{ ansible_distribution_release }}-updates"
      - "{{ ansible_distribution_release }}-backports"
    components:
      - "main"
      - "restricted"
      - "universe"
      - "multiverse"
    gpg_url: "https://mirror.local/repo-key.asc"
    signed_by: "/etc/apt/keyrings/local-ubuntu.asc"
    validate_certs: false
    username: "repo_user"                 # опционально
    password: "{{ vault_repo_pass }}"     # опционально, храните в Vault

# ===========================================
# Настройки для Oracle Linux (8 / 9)
# ===========================================
oraclelinux_repositories:
  - filename: "oracle-linux-ol{{ ansible_distribution_major_version }}.repo"
    username: "repo_user"                 # опционально для скачивания ключей
    password: "{{ vault_repo_pass }}"
    gpg_keys:
      - url: "https://mirror.local/{{ ansible_distribution_major_version }}/RPM-GPG-KEY-oracle"
        dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol{{ ansible_distribution_major_version }}"
        validate_certs: false
    repos:
      - id: "ol{{ ansible_distribution_major_version }}_baseos_latest"
        name: "Oracle Linux {{ ansible_distribution_major_version }} BaseOS Latest ($basearch)"
        baseurl: "https://mirror.local/{{ ansible_distribution_major_version }}/baseos/latest/$basearch/"
        gpgcheck: 1
        gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol{{ ansible_distribution_major_version }}"
        enabled: 1
        username: "repo_user"             # опционально для DNF
        password: "{{ vault_repo_pass }}"
      - id: "ol{{ ansible_distribution_major_version }}_appstream"
        name: "Oracle Linux {{ ansible_distribution_major_version }} Application Stream ($basearch)"
        baseurl: "https://mirror.local/{{ ansible_distribution_major_version }}/appstream/$basearch/"
        gpgcheck: 1
        gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol{{ ansible_distribution_major_version }}"
        enabled: 1
        username: "repo_user"
        password: "{{ vault_repo_pass }}"

  - filename: "uek-ol{{ ansible_distribution_major_version }}.repo"
    repos:
      - id: "ol{{ ansible_distribution_major_version }}_UEKR7"
        name: "Latest Unbreakable Enterprise Kernel Release 7 for Oracle Linux {{ ansible_distribution_major_version }} ($basearch)"
        baseurl: "https://mirror.local/{{ ansible_distribution_major_version }}/UEKR7/$basearch/"
        gpgcheck: 1
        gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol{{ ansible_distribution_major_version }}"
        enabled: 1
        username: "repo_user"
        password: "{{ vault_repo_pass }}"

  - filename: "virt-ol{{ ansible_distribution_major_version }}.repo"
    repos:
      - id: "ol{{ ansible_distribution_major_version }}_kvm_utils"
        name: "Oracle Linux {{ ansible_distribution_major_version }} KVM Utilities ($basearch)"
        baseurl: "https://mirror.local/{{ ansible_distribution_major_version }}/kvm/utils/$basearch/"
        gpgcheck: 1
        gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol{{ ansible_distribution_major_version }}"
        enabled: 0
```

handlers/main.yml

```yaml
---
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: true

- name: Update dnf cache
  ansible.builtin.dnf:
    update_cache: true
```

tasks/main.yml

```yaml
---
- name: Validate supported OS
  ansible.builtin.assert:
    that:
      - (ansible_distribution == "Ubuntu" and ansible_distribution_release in ["jammy", "noble"]) or
        (ansible_distribution == "OracleLinux" and ansible_distribution_major_version in ["8", "9"])
    fail_msg: "Unsupported OS. Supported: Ubuntu (Jammy, Noble) and Oracle Linux (8, 9)."

- name: Configure Ubuntu repositories
  ansible.builtin.include_tasks:
    file: ubuntu.yml
  when:
    - ansible_distribution == "Ubuntu"
    - ubuntu_repositories | length > 0

- name: Configure Oracle Linux repositories
  ansible.builtin.include_tasks:
    file: oraclelinux.yml
  when:
    - ansible_distribution == "OracleLinux"
    - oraclelinux_repositories | length > 0

- name: Flush handlers immediately
  ansible.builtin.meta: flush_handlers
```

tasks/ubuntu.yml

```yaml
---
- name: Ensure /etc/apt/keyrings directory exists
  ansible.builtin.file:
    path: /etc/apt/keyrings
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: Download GPG keys from authenticated local repository
  ansible.builtin.get_url:
    url: "{{ item.gpg_url }}"
    dest: "{{ item.signed_by }}"
    url_username: "{{ item.username | default(omit) }}"
    url_password: "{{ item.password | default(omit) }}"
    force_basic_auth: "{{ true if item.username is defined else omit }}"
    validate_certs: "{{ item.validate_certs | default(true) }}"
    force: true
    owner: root
    group: root
    mode: "0644"
  loop: "{{ ubuntu_repositories }}"
  loop_control:
    label: "{{ item.filename }}"
  when:
    - item.gpg_url is defined
    - item.signed_by is defined
  no_log: true

- name: Deploy APT authentication files
  ansible.builtin.template:
    src: "ubuntu.auth.conf.j2"
    dest: "/etc/apt/auth.conf.d/{{ item.filename | regex_replace('\\.sources$', '.conf') }}"
    owner: root
    group: root
    mode: "0600"
  loop: "{{ ubuntu_repositories }}"
  loop_control:
    label: "{{ item.filename }}"
  when:
    - item.username is defined
    - item.password is defined
  no_log: true

- name: Deploy Ubuntu DEB822 repository templates
  ansible.builtin.template:
    src: "ubuntu.sources.j2"
    dest: "/etc/apt/sources.list.d/{{ item.filename }}"
    owner: root
    group: root
    mode: "0644"
  loop: "{{ ubuntu_repositories }}"
  loop_control:
    label: "{{ item.filename }}"
  notify: Update apt cache
```

tasks/oraclelinux.yml

```yaml
---
- name: Download GPG keys from authenticated local repository for Oracle Linux
  ansible.builtin.get_url:
    url: "{{ item.1.url }}"
    dest: "{{ item.1.dest }}"
    url_username: "{{ item.1.username | default(item.0.username) | default(omit) }}"
    url_password: "{{ item.1.password | default(item.0.password) | default(omit) }}"
    force_basic_auth: "{{ true if (item.1.username is defined or item.0.username is defined) else omit }}"
    validate_certs: "{{ item.1.validate_certs | default(true) }}"
    force: true
    owner: root
    group: root
    mode: "0644"
  loop: "{{ oraclelinux_repositories | subelements('gpg_keys', {'skip_missing': true}) }}"
  loop_control:
    label: "{{ item.1.dest }}"
  no_log: true

- name: Import downloaded GPG keys into RPM database
  ansible.builtin.rpm_key:
    key: "{{ item.1.dest }}"
    state: present
  loop: "{{ oraclelinux_repositories | subelements('gpg_keys', {'skip_missing': true}) }}"
  loop_control:
    label: "{{ item.1.dest }}"

- name: Deploy Oracle Linux YUM/DNF repository templates
  ansible.builtin.template:
    src: "oraclelinux.repo.j2"
    dest: "/etc/yum.repos.d/{{ item.filename }}"
    owner: root
    group: root
    mode: "{{ '0600' if (item.repos | selectattr('password', 'defined') | list | length > 0) else '0644' }}"
  loop: "{{ oraclelinux_repositories }}"
  loop_control:
    label: "{{ item.filename }}"
  no_log: true
  notify: Update dnf cache
```

templates/ubuntu.sources.j2

```jinja2
# {{ ansible_managed }}
Types: {{ item.types | default('deb') }}
URIs: {{ item.uris }}
Suites: {{ item.suites | join(' ') }}
Components: {{ item.components | join(' ') }}
{% if item.signed_by is defined %}
Signed-By: {{ item.signed_by }}
{% elif item.trusted is defined %}
Trusted: {{ item.trusted }}
{% endif %}
```

templates/ubuntu.auth.conf.j2

```jinja2
# {{ ansible_managed }}
machine {{ item.uris | regex_replace('^https?://', '') }}
login {{ item.username }}
password {{ item.password }}
```

templates/oraclelinux.repo.j2

```jinja2
# {{ ansible_managed }}
{% for repo in item.repos %}
[{{ repo.id }}]
name={{ repo.name }}
baseurl={{ repo.baseurl }}
enabled={{ repo.enabled | default(1) }}
gpgcheck={{ repo.gpgcheck | default(0) }}
{% if repo.gpgkey is defined %}
gpgkey={{ repo.gpgkey }}
{% endif %}
{% if repo.username is defined and repo.password is defined %}
username={{ repo.username }}
password={{ repo.password }}
{% endif %}

{% endfor %}
```

Пример использования

```yaml
---
- name: Configure OS repositories
  hosts: all
  become: true
  roles:
    - role: os_repo
```

Примечания

· Пароль vault_repo_pass должен быть определён в зашифрованном виде (Ansible Vault).
· Для самоподписанных HTTPS-сертификатов используйте validate_certs: false и при необходимости добавьте корневой CA в доверенные на целевых хостах.
· Роль автоматически обновляет кэш пакетов (apt или dnf) только при изменении конфигурации репозиториев.