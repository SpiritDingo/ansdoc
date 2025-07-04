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