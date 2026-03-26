Ниже представлена Ansible роль для обновления операционных систем Oracle Linux, Ubuntu 22.04 и Ubuntu 24.04. Роль выполняет:

1. Проверку блокировок обновления:
   · Для Debian‑семейства (Ubuntu) – наличие активных процессов apt/dpkg, а также удерживаемых пакетов (apt-mark hold).
   · Для RedHat‑семейства (Oracle Linux) – наличие процессов yum/dnf и пакетов, заблокированных через yum versionlock.
2. Снятие блокировок:
   · Ожидание завершения запущенных менеджеров пакетов (до 5 минут).
   · Снятие удержаний пакетов в Ubuntu (apt-mark unhold).
   · Очистка versionlock в Oracle Linux (yum versionlock clear).
3. Обновление системы:
   · Обновление кэша пакетов.
   · Обновление всех установленных пакетов до последних версий.

Роль безопасна – удаление файлов блокировок не выполняется, только ожидание завершения процессов. Команды снятия удержаний выполняются только при их наличии.

Структура роли

```
os_update/
├── tasks/
│   ├── main.yml
│   ├── debian.yml
│   └── redhat.yml
└── meta/
    └── main.yml
```

Файлы роли

tasks/main.yml

```yaml
---
- name: Include OS‑specific variables (optional)
  include_vars: "{{ ansible_os_family }}.yml"
  ignore_errors: yes

- name: Update system for Debian family
  include_tasks: debian.yml
  when: ansible_os_family == "Debian"

- name: Update system for RedHat family
  include_tasks: redhat.yml
  when: ansible_os_family == "RedHat"
```

tasks/debian.yml

```yaml
---
- name: Check for running apt or dpkg processes
  ansible.builtin.command: pgrep -f "apt|dpkg"
  register: apt_processes
  changed_when: false
  failed_when: false

- name: Wait for package managers to finish
  ansible.builtin.wait_for:
    path: /var/lib/dpkg/lock
    state: absent
    timeout: 300
  when: apt_processes.stdout != ""
  ignore_errors: yes

- name: Get held packages
  ansible.builtin.command: apt-mark showhold
  register: held_packages
  changed_when: false
  failed_when: false

- name: Unhold packages if any
  ansible.builtin.command: apt-mark unhold {{ held_packages.stdout_lines | join(' ') }}
  when: held_packages.stdout != ""
  changed_when: true

- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600

- name: Upgrade all packages
  ansible.builtin.apt:
    upgrade: dist
    update_cache: no
```

tasks/redhat.yml

```yaml
---
- name: Check for running yum or dnf processes
  ansible.builtin.command: pgrep -f "yum|dnf"
  register: rpm_processes
  changed_when: false
  failed_when: false

- name: Wait for package managers to finish
  ansible.builtin.wait_for:
    path: /var/run/yum.pid
    state: absent
    timeout: 300
  when: rpm_processes.stdout != ""
  ignore_errors: yes

- name: Get versionlock packages
  ansible.builtin.command: yum versionlock list
  register: versionlock_packages
  changed_when: false
  failed_when: false
  ignore_errors: yes

- name: Clear versionlock if any
  ansible.builtin.command: yum versionlock clear
  when: versionlock_packages.stdout != ""
  changed_when: true
  ignore_errors: yes

- name: Update all packages
  ansible.builtin.yum:
    name: '*'
    state: latest
    update_cache: yes
```

meta/main.yml

```yaml
---
galaxy_info:
  author: your_name
  description: Update OS packages, removing update locks
  license: MIT
  min_ansible_version: "2.9"
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
        - noble
    - name: OracleLinux
      versions:
        - "7"
        - "8"
        - "9"
  galaxy_tags:
    - update
    - upgrade
    - security
    - apt
    - yum
    - dnf
```

Использование

Пример playbook:

```yaml
- hosts: all
  become: yes
  roles:
    - os_update
```

Пояснения

· Блокировки процессов: проверяется наличие запущенных apt/dpkg или yum/dnf. Если есть, роль ждёт исчезновения файла блокировки (/var/lib/dpkg/lock или /var/run/yum.pid). Таймаут 5 минут.
· Удерживаемые пакеты (Ubuntu): apt-mark showhold возвращает список. Если он не пуст, снимаем удержание через apt-mark unhold.
· Versionlock (Oracle Linux): yum versionlock list показывает заблокированные пакеты. Если есть, очищаем блокировку командой yum versionlock clear.
· Обновление: для Ubuntu используется apt upgrade dist (обновление дистрибутива), для Oracle Linux – yum update '*' (все пакеты). Модуль apt и yum идемпотентны, повторный запуск ничего не изменит, если все пакеты уже актуальны.

Роль можно расширить:

· Добавить перезагрузку при необходимости.
· Использовать модуль package_facts для определения менеджера пакетов.
· Обрабатывать отдельно Oracle Linux 8+ через dnf (модуль yum автоматически использует dnf, если он установлен).




Для добавления логирования результатов выполнения в файл на целевой системе, расширим роль следующими шагами:

1. Создадим каталог для логов (если его нет) – например, /var/log/ansible/.
2. В каждом таске обновления зарегистрируем вывод (stdout, stderr) в переменные.
3. После выполнения запишем содержимое переменных в лог-файл с временной меткой.

Логирование будет работать как для Debian (Ubuntu), так и для RedHat (Oracle Linux) семейств.

Обновлённая структура роли

```
os_update/
├── tasks/
│   ├── main.yml
│   ├── debian.yml
│   └── redhat.yml
├── vars/
│   └── main.yml
└── meta/
    └── main.yml
```

Файлы роли с добавленным логированием

tasks/main.yml

```yaml
---
- name: Include OS‑specific variables (optional)
  include_vars: "{{ ansible_os_family }}.yml"
  ignore_errors: yes

- name: Create log directory
  ansible.builtin.file:
    path: /var/log/ansible
    state: directory
    mode: '0755'
  become: yes

- name: Update system for Debian family
  include_tasks: debian.yml
  when: ansible_os_family == "Debian"

- name: Update system for RedHat family
  include_tasks: redhat.yml
  when: ansible_os_family == "RedHat"
```

tasks/debian.yml

```yaml
---
- name: Check for running apt or dpkg processes
  ansible.builtin.command: pgrep -f "apt|dpkg"
  register: apt_processes
  changed_when: false
  failed_when: false

- name: Wait for package managers to finish
  ansible.builtin.wait_for:
    path: /var/lib/dpkg/lock
    state: absent
    timeout: 300
  when: apt_processes.stdout != ""
  ignore_errors: yes

- name: Get held packages
  ansible.builtin.command: apt-mark showhold
  register: held_packages
  changed_when: false
  failed_when: false

- name: Unhold packages if any
  ansible.builtin.command: apt-mark unhold {{ held_packages.stdout_lines | join(' ') }}
  when: held_packages.stdout != ""
  changed_when: true
  register: unhold_result
  ignore_errors: yes

- name: Update apt cache
  ansible.builtin.apt:
    update_cache: yes
    cache_valid_time: 3600
  register: apt_update_result

- name: Upgrade all packages
  ansible.builtin.apt:
    upgrade: dist
    update_cache: no
  register: apt_upgrade_result

- name: Write update log
  ansible.builtin.copy:
    content: |
      ---
      Timestamp: {{ ansible_date_time.iso8601 }}
      Host: {{ inventory_hostname }}
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      
      === Held packages check ===
      {{ held_packages.stdout | default('No held packages') }}
      
      === Unhold operation ===
      {{ unhold_result.stdout | default('No unhold performed') }}
      {{ unhold_result.stderr | default('') }}
      
      === APT update ===
      {{ apt_update_result.stdout | default('') }}
      {{ apt_update_result.stderr | default('') }}
      
      === APT upgrade ===
      {{ apt_upgrade_result.stdout | default('') }}
      {{ apt_upgrade_result.stderr | default('') }}
    dest: "/var/log/ansible/os_update_{{ ansible_date_time.epoch }}.log"
    mode: '0644'
  become: yes
  when: apt_upgrade_result is defined
```

tasks/redhat.yml

```yaml
---
- name: Check for running yum or dnf processes
  ansible.builtin.command: pgrep -f "yum|dnf"
  register: rpm_processes
  changed_when: false
  failed_when: false

- name: Wait for package managers to finish
  ansible.builtin.wait_for:
    path: /var/run/yum.pid
    state: absent
    timeout: 300
  when: rpm_processes.stdout != ""
  ignore_errors: yes

- name: Get versionlock packages
  ansible.builtin.command: yum versionlock list
  register: versionlock_packages
  changed_when: false
  failed_when: false
  ignore_errors: yes

- name: Clear versionlock if any
  ansible.builtin.command: yum versionlock clear
  when: versionlock_packages.stdout != ""
  changed_when: true
  register: versionlock_clear_result
  ignore_errors: yes

- name: Update all packages
  ansible.builtin.yum:
    name: '*'
    state: latest
    update_cache: yes
  register: yum_upgrade_result

- name: Write update log
  ansible.builtin.copy:
    content: |
      ---
      Timestamp: {{ ansible_date_time.iso8601 }}
      Host: {{ inventory_hostname }}
      OS: {{ ansible_distribution }} {{ ansible_distribution_version }}
      
      === Versionlock packages ===
      {{ versionlock_packages.stdout | default('No versionlock entries') }}
      
      === Versionlock clear operation ===
      {{ versionlock_clear_result.stdout | default('No clear performed') }}
      {{ versionlock_clear_result.stderr | default('') }}
      
      === YUM/DNF upgrade ===
      {{ yum_upgrade_result.stdout | default('') }}
      {{ yum_upgrade_result.stderr | default('') }}
    dest: "/var/log/ansible/os_update_{{ ansible_date_time.epoch }}.log"
    mode: '0644'
  become: yes
  when: yum_upgrade_result is defined
```

meta/main.yml (без изменений)

```yaml
---
galaxy_info:
  author: your_name
  description: Update OS packages, removing update locks, with logging
  license: MIT
  min_ansible_version: "2.9"
  platforms:
    - name: Ubuntu
      versions:
        - focal
        - jammy
        - noble
    - name: OracleLinux
      versions:
        - "7"
        - "8"
        - "9"
  galaxy_tags:
    - update
    - upgrade
    - security
    - apt
    - yum
    - dnf
    - logging
```

Пояснения

· Логирование: после выполнения всех операций создаётся файл в /var/log/ansible/ с именем os_update_<timestamp>.log. В него записываются:
  · Временная метка и информация о хосте.
  · Состояние заблокированных пакетов (held или versionlock) и результат их снятия.
  · Вывод команд обновления кэша и апгрейда (stdout/stderr).
· Используется ansible_date_time.epoch для уникальности имени файла. Можно заменить на ansible_date_time.iso8601_basic или добавить переменную.
· Модуль copy с content позволяет записать многострочный текст без необходимости создавать отдельный файл-шаблон.
· Регистрация переменных (register) для каждой команды, чтобы потом использовать их вывод.
· Учтены случаи, когда некоторые операции не выполнялись (например, если не было удержанных пакетов) – используется фильтр default().

Использование

Playbook остаётся прежним:

```yaml
- hosts: all
  become: yes
  roles:
    - os_update
```

После выполнения роли на каждом хосте будет создан файл лога с подробной информацией о процессе обновления.

Дополнительные улучшения

· Можно добавить агрегированный лог на управляющем узле, используя ansible.builtin.fetch для сбора всех логов в центральное хранилище.
· Если необходимо дописывать в один и тот же файл (а не создавать новый каждый раз), замените copy на lineinfile или blockinfile, но тогда нужно контролировать размер файла.
· Для более структурированного логирования можно использовать JSON-формат и модуль ansible.builtin.copy с content: "{{ log_data | to_nice_json }}".

Этот подход обеспечивает полную прозрачность действий роли и упрощает отладку.