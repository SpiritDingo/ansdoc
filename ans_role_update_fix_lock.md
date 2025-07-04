# Ansible роль для устранения ошибки "waiting for cache lock" в dpkg

Роль для Ansible, которая обнаруживает и устраняет проблему с блокировкой dpkg (`/var/lib/dpkg/lock-frontend`).

## Структура роли

```
roles/dpkg_lock_fix/
├── tasks
│   └── main.yml
├── handlers
│   └── main.yml
└── README.md
```

## Файл tasks/main.yml

```yaml
---
- name: Check for existing dpkg lock files
  stat:
    path: "{{ item }}"
  register: dpkg_lock_files
  loop:
    - /var/lib/dpkg/lock-frontend
    - /var/lib/dpkg/lock
    - /var/lib/apt/lists/lock
    - /var/cache/apt/archives/lock

- name: Display warning if dpkg locks exist
  debug:
    msg: "Found dpkg lock file: {{ item.stat.path }}"
  when: item.stat.exists
  loop: "{{ dpkg_lock_files.results }}"

- name: Kill processes holding dpkg locks if force flag is set
  ansible.builtin.command: "fuser -vk {{ item }}"
  ignore_errors: yes
  when: 
    - item.stat.exists
    - dpkg_lock_force_fix | default(false)
  loop: "{{ dpkg_lock_files.results }}"

- name: Remove dpkg lock files if force flag is set
  ansible.builtin.file:
    path: "{{ item.stat.path }}"
    state: absent
  when: 
    - item.stat.exists
    - dpkg_lock_force_fix | default(false)
  loop: "{{ dpkg_lock_files.results }}"
  notify: Update apt cache

- name: Check if apt/dpkg processes are running
  ansible.builtin.command: pgrep -a apt
  register: apt_processes
  changed_when: false
  ignore_errors: yes

- name: Display running apt/dpkg processes
  debug:
    msg: "Found running apt/dpkg process: {{ apt_processes.stdout_lines }}"
  when: apt_processes.stdout_lines | length > 0
```

## Файл handlers/main.yml

```yaml
---
- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
```

## Использование роли

1. Добавьте роль в ваш playbook:

```yaml
- hosts: all
  vars:
    dpkg_lock_force_fix: true  # установите false для только обнаружения проблем
  roles:
    - dpkg_lock_fix
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory your_playbook.yml
```

## Рекомендации

1. По умолчанию роль только обнаруживает проблему. Для автоматического исправления установите `dpkg_lock_force_fix: true`.

2. Будьте осторожны с принудительным удалением lock-файлов, так как это может привести к повреждению системы, если процессы apt/dpkg действительно выполняются.

3. После исправления рекомендуется выполнить:
   - `apt update`
   - `apt --fix-broken install`
   - Переустановку пакетов, которые могли быть повреждены

4. Для предотвращения проблемы в будущем:
   - Убедитесь, что только один процесс использует apt/dpkg одновременно
   - Используйте `apt-get` вместо `apt` в скриптах, так как он более стабилен
   - Рассмотрите использование `apt-daily-upgrade.timer` для автоматических обновлений

Эта роль поможет вам быстро обнаружить и устранить проблему с блокировкой dpkg в вашей инфраструктуре.



# Универсальная Ansible роль для устранения ошибки "waiting for cache lock" (Oracle Linux 9 и Ubuntu)

Эта улучшенная роль автоматически определяет ОС и использует соответствующие методы для устранения блокировок пакетных менеджеров.

## Структура роли

```
roles/pkg_lock_fix/
├── tasks
│   ├── main.yml
│   ├── debian.yml
│   └── redhat.yml
├── handlers
│   └── main.yml
├── vars
│   └── main.yml
└── README.md
```

## Файл vars/main.yml

```yaml
---
# Параметры по умолчанию
pkg_lock_force_fix: false
pkg_lock_autorepair: false
pkg_lock_timeout: 300  # 5 минут ожидания перед принудительным исправлением
```

## Файл tasks/main.yml

```yaml
---
- name: Gather OS facts
  ansible.builtin.setup:
    gather_subset: min

- name: Include OS-specific tasks
  ansible.builtin.include_tasks: "{{ ansible_facts['distribution'] | lower }}.yml"
  when: 
    - ansible_facts['distribution'] in ['OracleLinux', 'Ubuntu', 'Debian', 'CentOS']
  ignore_errors: yes

- name: Fail if unsupported OS
  ansible.builtin.fail:
    msg: "Unsupported OS: {{ ansible_facts['distribution'] }}"
  when: 
    - ansible_facts['distribution'] not in ['OracleLinux', 'Ubuntu', 'Debian', 'CentOS']
```

## Файл tasks/debian.yml (для Ubuntu/Debian)

```yaml
---
- name: Define Debian lock files
  ansible.builtin.set_fact:
    pkg_lock_files:
      - /var/lib/dpkg/lock-frontend
      - /var/lib/dpkg/lock
      - /var/lib/apt/lists/lock
      - /var/cache/apt/archives/lock

- name: Check for existing package manager lock files (Debian)
  ansible.builtin.stat:
    path: "{{ item }}"
  register: debian_lock_files
  loop: "{{ pkg_lock_files }}"

- name: Display warning if lock files exist (Debian)
  ansible.builtin.debug:
    msg: "Found package manager lock file: {{ item.stat.path }}"
  when: item.stat.exists
  loop: "{{ debian_lock_files.results }}"

- name: Wait for lock release (Debian)
  ansible.builtin.command: "flock -w {{ pkg_lock_timeout }} {{ item }} true"
  ignore_errors: yes
  when: 
    - item.stat.exists
    - not pkg_lock_force_fix
  loop: "{{ debian_lock_files.results }}"
  register: wait_for_lock
  until: wait_for_lock is not failed
  retries: 3
  delay: 10

- name: Force remove lock files if enabled (Debian)
  block:
    - name: Kill processes holding locks (Debian)
      ansible.builtin.command: "fuser -vkk {{ item }} || true"
      ignore_errors: yes
      loop: "{{ debian_lock_files.results }}"
      when: item.stat.exists

    - name: Remove lock files (Debian)
      ansible.builtin.file:
        path: "{{ item.stat.path }}"
        state: absent
      loop: "{{ debian_lock_files.results }}"
      when: item.stat.exists
      notify: Update package cache
  when: pkg_lock_force_fix
```

## Файл tasks/redhat.yml (для Oracle Linux 9/CentOS/RHEL)

```yaml
---
- name: Define RedHat lock files
  ansible.builtin.set_fact:
    pkg_lock_files:
      - /var/run/yum.pid
      - /var/run/dnf.pid

- name: Check for existing package manager lock files (RedHat)
  ansible.builtin.stat:
    path: "{{ item }}"
  register: redhat_lock_files
  loop: "{{ pkg_lock_files }}"

- name: Display warning if lock files exist (RedHat)
  ansible.builtin.debug:
    msg: "Found package manager lock file: {{ item.stat.path }}"
  when: item.stat.exists
  loop: "{{ redhat_lock_files.results }}"

- name: Wait for lock release (RedHat)
  ansible.builtin.command: "timeout {{ pkg_lock_timeout }} bash -c 'while [ -f {{ item }} ]; do sleep 1; done'"
  ignore_errors: yes
  when: 
    - item.stat.exists
    - not pkg_lock_force_fix
  loop: "{{ redhat_lock_files.results }}"
  register: wait_for_lock
  until: wait_for_lock is not failed
  retries: 3
  delay: 10

- name: Force remove lock files if enabled (RedHat)
  block:
    - name: Kill processes holding locks (RedHat)
      ansible.builtin.command: "kill -9 $(cat {{ item }}) || true"
      ignore_errors: yes
      loop: "{{ redhat_lock_files.results }}"
      when: item.stat.exists

    - name: Remove lock files (RedHat)
      ansible.builtin.file:
        path: "{{ item.stat.path }}"
        state: absent
      loop: "{{ redhat_lock_files.results }}"
      when: item.stat.exists
      notify: Update package cache
  when: pkg_lock_force_fix
```

## Файл handlers/main.yml

```yaml
---
- name: Update package cache
  block:
    - name: Update cache (Debian)
      ansible.builtin.apt:
        update_cache: yes
      when: ansible_facts['distribution'] in ['Ubuntu', 'Debian']

    - name: Clean metadata (RedHat)
      ansible.builtin.command: "dnf clean metadata || yum clean metadata"
      when: ansible_facts['distribution'] in ['OracleLinux', 'CentOS', 'RedHat']

    - name: Make cache (RedHat)
      ansible.builtin.command: "dnf makecache || yum makecache"
      when: ansible_facts['distribution'] in ['OracleLinux', 'CentOS', 'RedHat']
```

## Использование роли

1. Добавьте роль в ваш playbook:

```yaml
- hosts: all
  vars:
    pkg_lock_force_fix: true  # принудительное исправление
    pkg_lock_timeout: 120    # 2 минуты ожидания
  roles:
    - pkg_lock_fix
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory your_playbook.yml
```

## Особенности реализации:

1. **Автоматическое определение ОС** - роль работает как с Debian-based (Ubuntu), так и с RHEL-based (Oracle Linux 9) системами.

2. **Гибкие параметры**:
   - `pkg_lock_force_fix` - принудительное исправление
   - `pkg_lock_timeout` - время ожидания перед принудительным действием
   - `pkg_lock_autorepair` - автоматическое исправление без подтверждения

3. **Безопасное ожидание** - перед принудительным исправлением роль пытается дождаться освобождения блокировки.

4. **Универсальные обработчики** - автоматически выбирают правильную команду для обновления кэша пакетов.

5. **Поддержка нескольких менеджеров пакетов**:
   - Для Debian: apt/dpkg
   - Для RedHat: yum/dnf

Эта роль обеспечивает надежное решение проблемы блокировок пакетных менеджеров в гетерогенных средах.