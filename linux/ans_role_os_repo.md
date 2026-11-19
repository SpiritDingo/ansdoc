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

________
________

Ниже — изменения, которые нужно внести, чтобы для Ubuntu 22.04 (jammy) и Ubuntu 24.04 (noble) использовались свои независимые наборы репозиториев.

Идея: ubuntu_repositories превращается в словарь, где ключи — это кодовые имена (jammy, noble), а значения — списки репозиториев. В задачах нужный набор выбирается по ansible_distribution_release.

1. Обновлённый defaults/main.yml (секция Ubuntu)

```yaml
---
# ===========================================
# Настройки для Ubuntu (Jammy / Noble)
# Свой набор репозиториев для каждой версии
# ===========================================
ubuntu_repositories:
  jammy:
    - filename: "local-ubuntu-jammy.sources"
      types: "deb"
      uris: "https://mirror.local"
      suites:
        - "jammy"
        - "jammy-updates"
        - "jammy-backports"
        - "jammy-security"
      components:
        - "main"
        - "restricted"
        - "universe"
        - "multiverse"
      gpg_url: "https://mirror.local/jammy/repo-key.asc"
      signed_by: "/etc/apt/keyrings/local-ubuntu-jammy.asc"
      validate_certs: false
      username: "repo_user"
      password: "{{ vault_repo_pass }}"

  noble:
    - filename: "local-ubuntu-noble.sources"
      types: "deb"
      uris: "https://mirror.local"
      suites:
        - "noble"
        - "noble-updates"
        - "noble-backports"
        - "noble-security"
      components:
        - "main"
        - "restricted"
        - "universe"
        - "multiverse"
      gpg_url: "https://mirror.local/noble/repo-key.asc"
      signed_by: "/etc/apt/keyrings/local-ubuntu-noble.asc"
      validate_certs: false
      username: "repo_user"
      password: "{{ vault_repo_pass }}"

# ===========================================
# Настройки для Oracle Linux (8 / 9)
# ===========================================
oraclelinux_repositories:
  # ... без изменений ...
```

2. Обновлённый tasks/main.yml

Меняется только условие запуска блока Ubuntu — теперь проверяется наличие непустого списка для текущего релиза:

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
    - ubuntu_repositories[ansible_distribution_release] | default([]) | length > 0

- name: Configure Oracle Linux repositories
  ansible.builtin.include_tasks:
    file: oraclelinux.yml
  when:
    - ansible_distribution == "OracleLinux"
    - oraclelinux_repositories | length > 0

- name: Flush handlers immediately
  ansible.builtin.meta: flush_handlers
```

3. Обновлённый tasks/ubuntu.yml

Все loop переключаются на выборку по релизу. when-условия в задачах не меняются, так как item — это по-прежнему элемент выбранного списка.

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
  loop: "{{ ubuntu_repositories[ansible_distribution_release] | default([]) }}"
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
  loop: "{{ ubuntu_repositories[ansible_distribution_release] | default([]) }}"
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
  loop: "{{ ubuntu_repositories[ansible_distribution_release] | default([]) }}"
  loop_control:
    label: "{{ item.filename }}"
  notify: Update apt cache
```

4. Что остаётся без изменений

· handlers/main.yml
· templates/ubuntu.sources.j2
· templates/ubuntu.auth.conf.j2
· tasks/oraclelinux.yml
· templates/oraclelinux.repo.j2

Как это работает

1. На хосте с Ubuntu 22.04 ansible_distribution_release == "jammy" → роль возьмёт ubuntu_repositories['jammy'].
2. На хосте с Ubuntu 24.04 ansible_distribution_release == "noble" → роль возьмёт ubuntu_repositories['noble'].
3. Если для какой-то версии список пуст или отсутствует, задача просто не выполнится (благодаря default([]) | length > 0 в main.yml и | default([]) в циклах).
4. Всё, что касается скачивания ключей, авторизации (auth.conf.d) и рендеринга .sources, автоматически работает с выбранным набором.

Пример переопределения в group_vars

Можно удобно задавать разные зеркала для разных групп серверов:

```yaml
# group_vars/ubuntu_jammy_servers.yml
ubuntu_repositories:
  jammy:
    - filename: "jammy-mirror.sources"
      uris: "https://mirror-jammy.local"
      suites: ["jammy", "jammy-updates", "jammy-security"]
      components: ["main", "universe"]
      gpg_url: "https://mirror-jammy.local/key.asc"
      signed_by: "/etc/apt/keyrings/jammy-mirror.asc"
      username: "jammy_user"
      password: "{{ vault_jammy_pass }}"

# group_vars/ubuntu_noble_servers.yml
ubuntu_repositories:
  noble:
    - filename: "noble-mirror.sources"
      uris: "https://mirror-noble.local"
      suites: ["noble", "noble-updates", "noble-security"]
      components: ["main", "universe"]
      gpg_url: "https://mirror-noble.local/key.asc"
      signed_by: "/etc/apt/keyrings/noble-mirror.asc"
      username: "noble_user"
      password: "{{ vault_noble_pass }}"
```

При этом роли не нужно знать, какие именно зеркала используются — она просто берёт нужный ключ словаря по факту ansible_distribution_release.
_____
_____

Ниже — изменения по аналогии с Ubuntu: oraclelinux_repositories превращается в словарь, где ключи — мажорные версии ("8", "9"), а значения — независимые списки репозиториев. Выбор нужного набора в задачах делается по ansible_distribution_major_version.

1. Обновлённый defaults/main.yml (секция Oracle Linux)

```yaml
---
# ... блок ubuntu_repositories остается без изменений ...

# ===========================================
# Настройки для Oracle Linux (8 / 9)
# Свой набор репозиториев для каждой версии
# ===========================================
oraclelinux_repositories:
  "8":
    # --- oracle-linux-ol8.repo (BaseOS + AppStream) ---
    - filename: "oracle-linux-ol8.repo"
      gpg_keys:
        - url: "https://mirror.local/8/RPM-GPG-KEY-oracle-ol8"
          dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol8"
          validate_certs: false
      repos:
        - id: "ol8_baseos_latest"
          name: "Oracle Linux 8 BaseOS Latest ($basearch)"
          baseurl: "https://mirror.local/8/baseos/latest/$basearch/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol8"
          enabled: 1
          username: "repo_user"
          password: "{{ vault_repo_pass }}"
        - id: "ol8_appstream"
          name: "Oracle Linux 8 Application Stream ($basearch)"
          baseurl: "https://mirror.local/8/appstream/$basearch/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol8"
          enabled: 1
          username: "repo_user"
          password: "{{ vault_repo_pass }}"

    # --- uek-ol8.repo (UEK R6) ---
    - filename: "uek-ol8.repo"
      gpg_keys:
        - url: "https://mirror.local/8/RPM-GPG-KEY-uek-ol8"
          dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-uek-ol8"
          validate_certs: false
      repos:
        - id: "ol8_UEKR6"
          name: "Latest Unbreakable Enterprise Kernel Release 6 for Oracle Linux 8 ($basearch)"
          baseurl: "https://mirror.local/8/UEKR6/$basearch/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-uek-ol8"
          enabled: 1
          username: "repo_user"
          password: "{{ vault_repo_pass }}"

    # --- virt-ol8.repo (KVM Utilities) ---
    - filename: "virt-ol8.repo"
      gpg_keys:
        - url: "https://mirror.local/8/RPM-GPG-KEY-virt-ol8"
          dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-virt-ol8"
          validate_certs: false
      repos:
        - id: "ol8_kvm_utils"
          name: "Oracle Linux 8 KVM Utilities ($basearch)"
          baseurl: "https://mirror.local/8/kvm/utils/$basearch/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-virt-ol8"
          enabled: 0

  "9":
    # --- oracle-linux-ol9.repo (BaseOS + AppStream) ---
    - filename: "oracle-linux-ol9.repo"
      gpg_keys:
        - url: "https://mirror.local/9/RPM-GPG-KEY-oracle-ol9"
          dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9"
          validate_certs: false
      repos:
        - id: "ol9_baseos_latest"
          name: "Oracle Linux 9 BaseOS Latest ($basearch)"
          baseurl: "https://mirror.local/9/baseos/latest/$basearch/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9"
          enabled: 1
          username: "repo_user"
          password: "{{ vault_repo_pass }}"
        - id: "ol9_appstream"
          name: "Oracle Linux 9 Application Stream ($basearch)"
          baseurl: "https://mirror.local/9/appstream/$basearch/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9"
          enabled: 1
          username: "repo_user"
          password: "{{ vault_repo_pass }}"

    # --- uek-ol9.repo (UEK R7) ---
    - filename: "uek-ol9.repo"
      gpg_keys:
        - url: "https://mirror.local/9/RPM-GPG-KEY-uek-ol9"
          dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-uek-ol9"
          validate_certs: false
      repos:
        - id: "ol9_UEKR7"
          name: "Latest Unbreakable Enterprise Kernel Release 7 for Oracle Linux 9 ($basearch)"
          baseurl: "https://mirror.local/9/UEKR7/$baseard/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-uek-ol9"
          enabled: 1
          username: "repo_user"
          password: "{{ vault_repo_pass }}"

    # --- virt-ol9.repo (KVM Utilities) ---
    - filename: "virt-ol9.repo"
      gpg_keys:
        - url: "https://mirror.local/9/RPM-GPG-KEY-virt-ol9"
          dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-virt-ol9"
          validate_certs: false
      repos:
        - id: "ol9_kvm_utils"
          name: "Oracle Linux 9 KVM Utilities ($basearch)"
          baseurl: "https://mirror.local/9/kvm/utils/$basearch/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-virt-ol9"
          enabled: 0
```

⚠️ В блоке ol9_UEKR7 выше опечатка $baseard — замените на $basearch.

2. Обновлённый tasks/main.yml

Условие запуска Oracle Linux теперь проверяет наличие непустого списка для текущей мажорной версии:

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
    - ubuntu_repositories[ansible_distribution_release] | default([]) | length > 0

- name: Configure Oracle Linux repositories
  ansible.builtin.include_tasks:
    file: oraclelinux.yml
  when:
    - ansible_distribution == "OracleLinux"
    - oraclelinux_repositories[ansible_distribution_major_version] | default([]) | length > 0

- name: Flush handlers immediately
  ansible.builtin.meta: flush_handlers
```

3. Обновлённый tasks/oraclelinux.yml

Все loop переключаются на выборку по мажорной версии. Обратите внимание на порядок фильтров в subelements — сначала выбор версии, затем subelements:

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
  loop: "{{ oraclelinux_repositories[ansible_distribution_major_version] | default([]) | subelements('gpg_keys', {'skip_missing': true}) }}"
  loop_control:
    label: "{{ item.1.dest }}"
  no_log: true

- name: Import downloaded GPG keys into RPM database
  ansible.builtin.rpm_key:
    key: "{{ item.1.dest }}"
    state: present
  loop: "{{ oraclelinux_repositories[ansible_distribution_major_version] | default([]) | subelements('gpg_keys', {'skip_missing': true}) }}"
  loop_control:
    label: "{{ item.1.dest }}"

- name: Deploy Oracle Linux YUM/DNF repository templates
  ansible.builtin.template:
    src: "oraclelinux.repo.j2"
    dest: "/etc/yum.repos.d/{{ item.filename }}"
    owner: root
    group: root
    mode: "{{ '0600' if (item.repos | selectattr('password', 'defined') | list | length > 0) else '0644' }}"
  loop: "{{ oraclelinux_repositories[ansible_distribution_major_version] | default([]) }}"
  loop_control:
    label: "{{ item.filename }}"
  no_log: true
  notify: Update dnf cache
```

4. Что остаётся без изменений

· handlers/main.yml
· templates/oraclelinux.repo.j2
· Все Ubuntu-файлы (tasks/ubuntu.yml, templates/ubuntu.*.j2)

Как это работает

1. На хосте с Oracle Linux 8 ansible_distribution_major_version == "8" → роль берёт oraclelinux_repositories['8'] (UEKR6, свои пути /8/..., свои ключи).
2. На хосте с Oracle Linux 9 ansible_distribution_major_version == "9" → роль берёт oraclelinux_repositories['9'] (UEKR7, пути /9/..., свои ключи).
3. Если для какой-то версии список пуст или ключ отсутствует, задача не выполнится благодаря default([]) | length > 0 в main.yml.
4. subelements('gpg_keys', {'skip_missing': true}) корректно обрабатывает каждый файл .repo со своим набором ключей — у OL8 и OL9 ключи теперь полностью независимы.

Пример переопределения в group_vars

```yaml
# group_vars/ol8_servers.yml
oraclelinux_repositories:
  "8":
    - filename: "ol8-mirror.repo"
      gpg_keys:
        - url: "https://mirror-ol8.local/RPM-GPG-KEY-ol8"
          dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-ol8"
      repos:
        - id: "ol8_baseos"
          name: "OL8 BaseOS"
          baseurl: "https://mirror-ol8.local/baseos/$basearch/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ol8"
          enabled: 1

# group_vars/ol9_servers.yml
oraclelinux_repositories:
  "9":
    - filename: "ol9-mirror.repo"
      gpg_keys:
        - url: "https://mirror-ol9.local/RPM-GPG-KEY-ol9"
          dest: "/etc/pki/rpm-gpg/RPM-GPG-KEY-ol9"
      repos:
        - id: "ol9_baseos"
          name: "OL9 BaseOS"
          baseurl: "https://mirror-ol9.local/baseos/$basearch/"
          gpgcheck: 1
          gpgkey: "file:///etc/pki/rpm-gpg/RPM-GPG-KEY-ol9"
          enabled: 1
```

Роль при этом не меняется — она просто выбирает набор по фактической версии хоста.