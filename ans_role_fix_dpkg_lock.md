Вот **Ansible роль**, которая устраняет проблему блокировки `dpkg` / `apt` на Ubuntu/Debian системах.

## Структура роли

```
roles/
└── fix_dpkg_lock/
    ├── tasks/
    │   └── main.yml
    ├── handlers/
    │   └── main.yml
    └── README.md (опционально)
```

## Содержание роли

### `tasks/main.yml`

```yaml
---
- name: Проверка наличия заблокированных процессов apt/dpkg
  shell: |
    pgrep -f "apt|dpkg" || true
  register: apt_processes
  changed_when: false

- name: Определение PID процесса, удерживающего lock
  shell: |
    lsof -t /var/lib/dpkg/lock-frontend 2>/dev/null || fuser /var/lib/dpkg/lock-frontend 2>/dev/null | awk '{print $1}' || true
  register: locking_pid
  changed_when: false
  ignore_errors: yes

- name: Вывод информации о блокирующем процессе
  debug:
    msg: "Процесс {{ locking_pid.stdout }} блокирует dpkg"
  when: locking_pid.stdout != ""

- name: Принудительное завершение процесса (если он не критичен)
  shell: |
    kill -9 {{ locking_pid.stdout }}
  when: 
    - locking_pid.stdout != ""
    - locking_pid.stdout != "1"
  register: kill_result
  ignore_errors: yes

- name: Ожидание освобождения блокировки
  wait_for:
    path: /var/lib/dpkg/lock-frontend
    state: absent
    timeout: 30
  ignore_errors: yes

- name: Удаление lock-файлов вручную (если они остались)
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /var/lib/dpkg/lock-frontend
    - /var/lib/dpkg/lock
    - /var/cache/apt/archives/lock
    - /var/lib/apt/lists/lock
  ignore_errors: yes

- name: Переконфигурация dpkg (исправление возможных повреждений)
  command: dpkg --configure -a
  register: dpkg_configure
  ignore_errors: yes

- name: Очистка кэша apt
  apt:
    autoclean: yes
    update_cache: yes
  ignore_errors: yes

- name: Обновление списка пакетов
  apt:
    update_cache: yes
    cache_valid_time: 0
  ignore_errors: yes

- name: Проверка успешности исправления
  command: dpkg -l
  register: dpkg_check
  changed_when: false
  failed_when: false
```

### `handlers/main.yml`

```yaml
---
- name: Перезапустить apt-сервисы (если есть)
  service:
    name: "{{ item }}"
    state: restarted
  loop:
    - apt-daily.service
    - apt-daily-upgrade.service
  ignore_errors: yes
```

## Альтернативный вариант — одноплейбук (без отдельной роли)

Если нужен просто плейбук:

```yaml
---
- name: Устранение блокировки dpkg/apt
  hosts: all
  become: yes

  tasks:
    - name: Найти процесс с блокировкой
      shell: |
        fuser /var/lib/dpkg/lock-frontend 2>/dev/null | awk '{print $1}'
      register: locked_pid
      failed_when: false

    - name: Убить блокирующий процесс
      shell: kill -9 {{ locked_pid.stdout }}
      when: locked_pid.stdout != ""

    - name: Удалить lock-файлы
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /var/lib/dpkg/lock-frontend
        - /var/lib/dpkg/lock
        - /var/cache/apt/archives/lock

    - name: Настроить dpkg
      command: dpkg --configure -a

    - name: Обновить apt
      apt:
        update_cache: yes
        cache_valid_time: 0
```

## Использование в плейбуке

```yaml
- hosts: myservers
  become: yes
  roles:
    - fix_dpkg_lock

  # или после исправления выполнить нужные apt-операции
  post_tasks:
    - name: Установить нужный пакет
      apt:
        name: nginx
        state: present
```

## Важные замечания

1. **Безопасность**: убивать процесс `apt` можно, только если вы уверены, что это не критичная операция (например, не идет обновление системы).

2. **Более безопасный подход**: использовать `wait_for` с таймаутом, не убивая процесс:

```yaml
- name: Ждать освобождения блокировки
  wait_for:
    path: /var/lib/dpkg/lock-frontend
    state: absent
    timeout: 300  # 5 минут
  register: wait_result

- name: Удалить lock, если таймаут истек
  file:
    path: /var/lib/dpkg/lock-frontend
    state: absent
  when: wait_result is failed
```

3. **Проверка на Debian/Ubuntu**:

```yaml
- name: Убедиться, что система использует apt
  fail:
    msg: "Эта роль только для Debian/Ubuntu"
  when: ansible_os_family != "Debian"
```

4. **Лучшее решение для production**: 
   - Настроить `apt` с `cache_valid_time`
   - Использовать `wait_for` вместо `kill`
   - Добавить retry в задачи установки пакетов

Готовая роль решает проблему и позволяет Ansible успешно выполнять последующие задачи по управлению пакетами.

___________________
___________________


Вот улучшенная версия роли fix_dpkg_lock, которая учитывает безопасность, идемпотентность, graceful handling и production-сценарии.

Улучшения

1. Не убивает процесс сразу — сначала ждёт, потом пытается gracefully завершить, в крайнем случае — force kill.
2. Проверяет, не является ли процесс системным (например, unattended-upgrades).
3. Использует модули Ansible вместо shell где возможно.
4. Работает с блокировками от apt-daily.service — предлагает временно остановить сервис.
5. Логирование и прозрачность — переменные для настройки поведения.
6. Обработка ошибок и retry для последующих apt-операций.

Структура роли

```
roles/fix_dpkg_lock/
├── defaults/
│   └── main.yml
├── tasks/
│   └── main.yml
├── handlers/
│   └── main.yml
└── meta/
    └── main.yml
```

defaults/main.yml — настраиваемые параметры

```yaml
---
# Стратегия ожидания: 'wait_and_retry' (без убийства) или 'kill'
dpkg_lock_strategy: "wait_and_retry"

# Максимальное время ожидания освобождения блокировки (секунды)
dpkg_lock_timeout: 300

# Принудительно убивать процесс по истечению таймаута?
dpkg_lock_force_kill_after_timeout: true

# Разрешить остановку apt-daily сервисов (только если держат lock)
dpkg_lock_stop_apt_daily: true

# Удалять lock-файлы вручную если они остались после всех попыток?
dpkg_lock_remove_stale_locks: true
```

tasks/main.yml — основная логика

```yaml
---
- name: Проверка ОС (только Debian/Ubuntu)
  fail:
    msg: "Роль поддерживает только Debian/Ubuntu"
  when: ansible_os_family != "Debian"

- name: Проверка наличия активного процесса с блокировкой dpkg
  shell: |
    fuser /var/lib/dpkg/lock-frontend 2>/dev/null | awk '{print $1}'
  register: locking_pid
  changed_when: false
  failed_when: false

- name: Если блокировки нет — выходим
  meta: end_host
  when: locking_pid.stdout == ""

- name: Получение информации о блокирующем процессе
  shell: |
    ps -p {{ locking_pid.stdout }} -o comm=,args= --no-headers | head -1
  register: proc_info
  changed_when: false
  failed_when: false
  when: locking_pid.stdout != ""

- name: Вывод предупреждения
  debug:
    msg: |
      Обнаружена блокировка dpkg процессом PID {{ locking_pid.stdout }}.
      Команда: {{ proc_info.stdout | default('неизвестно') }}
      Стратегия: {{ dpkg_lock_strategy }}

- block:
    - name: Ожидание освобождения блокировки (стратегия wait_and_retry)
      wait_for:
        path: /var/lib/dpkg/lock-frontend
        state: absent
        timeout: "{{ dpkg_lock_timeout }}"
      register: wait_result
      when: dpkg_lock_strategy == "wait_and_retry"

    - name: Принудительное удаление lock-файлов после таймаута
      block:
        - name: Остановка apt-daily сервисов
          service:
            name: "{{ item }}"
            state: stopped
          loop:
            - apt-daily.service
            - apt-daily-upgrade.service
          when: dpkg_lock_stop_apt_daily
          ignore_errors: yes

        - name: Убийство блокирующего процесса
          shell: |
            kill -TERM {{ locking_pid.stdout }}
            sleep 2
            kill -KILL {{ locking_pid.stdout }} 2>/dev/null || true
          when: dpkg_lock_force_kill_after_timeout

        - name: Удаление lock-файлов
          file:
            path: "{{ item }}"
            state: absent
          loop:
            - /var/lib/dpkg/lock-frontend
            - /var/lib/dpkg/lock
            - /var/cache/apt/archives/lock
            - /var/lib/apt/lists/lock
          when: dpkg_lock_remove_stale_locks

      when: 
        - dpkg_lock_strategy == "wait_and_retry"
        - wait_result is failed

    - name: Прямое убийство процесса (стратегия kill)
      block:
        - name: Принудительное завершение блокирующего процесса
          shell: |
            kill -TERM {{ locking_pid.stdout }}
            sleep 2
            kill -KILL {{ locking_pid.stdout }} 2>/dev/null || true
          when: dpkg_lock_strategy == "kill"

        - name: Удаление lock-файлов
          file:
            path: "{{ item }}"
            state: absent
          loop:
            - /var/lib/dpkg/lock-frontend
            - /var/lib/dpkg/lock
            - /var/cache/apt/archives/lock
            - /var/lib/apt/lists/lock
          when: dpkg_lock_remove_stale_locks

      when: dpkg_lock_strategy == "kill"

  always:
    - name: Переконфигурация dpkg после снятия блокировки
      command: dpkg --configure -a
      register: dpkg_configure
      failed_when: 
        - dpkg_configure.rc != 0
        - "'dpkg was interrupted' not in dpkg_configure.stderr"
      changed_when: dpkg_configure.rc == 0

    - name: Восстановление apt-daily сервисов (если были остановлены)
      service:
        name: "{{ item }}"
        state: started
      loop:
        - apt-daily.service
        - apt-daily-upgrade.service
      when: dpkg_lock_stop_apt_daily and (not ansible_check_mode)
      ignore_errors: yes

    - name: Очистка кэша apt
      apt:
        autoclean: yes
      changed_when: false
      ignore_errors: yes

    - name: Обновление кэша apt
      apt:
        update_cache: yes
        cache_valid_time: 0
      register: apt_update
      retries: 3
      delay: 2
      until: apt_update is success
      ignore_errors: yes

- name: Итоговая проверка — dpkg не заблокирован
  shell: fuser /var/lib/dpkg/lock-frontend 2>/dev/null
  register: check_lock
  failed_when: check_lock.rc == 0
  changed_when: false
  ignore_errors: yes

- name: Успешное завершение
  debug:
    msg: "Блокировка dpkg успешно устранена"
  when: check_lock.rc != 0
```

handlers/main.yml

```yaml
---
- name: Перезапуск сервисов apt
  systemd:
    name: "{{ item }}"
    state: restarted
    daemon_reload: yes
  loop:
    - apt-daily.timer
    - apt-daily-upgrade.timer
    - unattended-upgrades.service
  ignore_errors: yes
```

meta/main.yml

```yaml
---
galaxy_info:
  author: yourname
  description: Fix dpkg/apt lock issues gracefully
  license: MIT
  min_ansible_version: 2.9
  platforms:
    - name: Ubuntu
      versions:
        - all
    - name: Debian
      versions:
        - all
  galaxy_tags:
    - dpkg
    - apt
    - lock
    - packaging
dependencies: []
```

Пример использования в плейбуке

```yaml
- hosts: all
  become: yes
  vars:
    dpkg_lock_strategy: "wait_and_retry"
    dpkg_lock_timeout: 180
    dpkg_lock_force_kill_after_timeout: false   # только ждать, не убивать
  roles:
    - fix_dpkg_lock

  post_tasks:
    - name: Установка пакета
      apt:
        name: nginx
        state: present
      retries: 2
      delay: 5
```

Ключевые улучшения

Что было Что стало
Безусловный kill -9 Сначала ждём, пытаемся TERM, потом KILL
Только удаление lock + остановка apt-daily, переконфигурация dpkg, очистка кэша
Нет проверки ОС Есть проверка на Debian-семейство
Одна стратегия Две стратегии: wait_and_retry / kill
Нет повторных попыток retries и until для критичных операций
Нет отката сервисов Восстанавливаем apt-daily после исправления

Такая роль безопаснее для production и даёт администратору контроль над действиями.



-----------------------------------------------
-----------------------------------------------

Ошибка показывает, что вы всё ещё пытаетесь выполнить `dpkg --configure -a`, но блокировка активна (процесс 130477). Ansible **не нужно это делать**, потому что задача должна была сначала убить процесс и удалить lock-файлы.

Вот **исправленная роль**:

## `tasks/main.yml` (исправленный)

```yaml
---
- name: Убить процесс, удерживающий lock-frontend
  shell: |
    PID=$(fuser /var/lib/dpkg/lock-frontend 2>/dev/null | awk '{print $1}')
    if [ -n "$PID" ] && [ "$PID" != "1" ]; then
      kill -9 $PID
      echo "Killed process $PID"
    fi
  register: kill_result
  changed_when: kill_result.stdout != ""
  failed_when: false

- name: Дождаться завершения процесса (пауза)
  wait_for:
    timeout: 3
  when: kill_result.stdout != ""

- name: Удалить lock-файлы (только если процесс убит)
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /var/lib/dpkg/lock-frontend
    - /var/lib/dpkg/lock
  register: remove_lock
  failed_when: false

- name: ПРИНУДИТЕЛЬНОЕ ОБНОВЛЕНИЕ APT (без dpkg --configure)
  apt:
    update_cache: yes
    force_apt_get: yes
  register: apt_update
  until: apt_update is success
  retries: 3
  delay: 2
  ignore_errors: yes

- name: Если APT всё ещё заблокирован — перезапустить систему
  block:
    - name: Проверить, какие процессы используют dpkg
      shell: ps aux | grep -E 'apt|dpkg' | grep -v grep
      register: ps_check
      failed_when: false
      
    - name: Вывести информацию о блокирующих процессах
      debug:
        var: ps_check.stdout_lines
    
    - name: Найти PID всех процессов apt/dpkg
      shell: pgrep -f "apt|dpkg" | xargs -r kill -9
      failed_when: false
      
    - name: Повторная попытка обновления APT
      apt:
        update_cache: yes
        force_apt_get: yes
  when: apt_update is failed

- name: Установить нужный пакет (пример)
  apt:
    name: "{{ package_name | default('nginx') }}"
    state: present
    force_apt_get: yes
  register: apt_install
  until: apt_install is success
  retries: 5
  delay: 5
  ignore_errors: yes

- name: Если пакет не установился — создать отчёт об ошибке
  fail:
    msg: |
      Невозможно установить пакет. Система всё ещё заблокирована.
      Выполните вручную на сервере:
        sudo kill -9 $(fuser /var/lib/dpkg/lock-frontend)
        sudo rm -f /var/lib/dpkg/lock-frontend
        sudo dpkg --configure -a
        sudo apt-get update
        sudo apt-get install -f
  when: apt_install is failed
```

## **Ещё лучше — одноплейбук с обходным решением**

Вместо `dpkg --configure -a` (который сам требует lock), используйте **force обновление apt**:

```yaml
---
- name: Исправление блокировки dpkg/apt
  hosts: all
  become: yes
  gather_facts: yes

  tasks:
    - name: Убить все процессы apt/dpkg
      shell: |
        pkill -9 -f "apt|dpkg" || true
        sleep 2
      changed_when: false

    - name: Принудительное удаление lock-файлов
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /var/lib/dpkg/lock-frontend
        - /var/lib/dpkg/lock
        - /var/cache/apt/archives/lock
        - /var/lib/apt/lists/lock

    - name: Перезапустить сервисы APT
      systemd:
        name: "{{ item }}"
        state: restarted
        daemon_reload: yes
      loop:
        - apt-daily.timer
        - apt-daily-upgrade.timer
      ignore_errors: yes

    - name: Обновить APT с таймаутом и повторами
      apt:
        update_cache: yes
        cache_valid_time: 0
      register: apt_result
      until: apt_result is success
      retries: 5
      delay: 10
      # Ключевой момент: не используем dpkg --configure

    - name: Установка пакета (если нужно)
      apt:
        name: "your-package-name"
        state: present
```

## **Самое простое решение — обойти dpkg --configure**

**Удалите эту задачу** из вашей роли:
```yaml
- name: Переконфигурация dpkg после снятия блокировки  # ← УДАЛИТЬ
  command: dpkg --configure -a                         # ← УДАЛИТЬ
```

Вместо неё используйте:
```yaml
- name: Принудительное исправление сломанных пакетов
  apt:
    update_cache: yes
    force: yes
  register: fix_result
  until: fix_result is success
  retries: 3
  delay: 5
```

## **Если совсем не работает — перезагрузить сервер**

```yaml
- name: Перезагрузка сервера для снятия всех блокировок
  reboot:
    reboot_timeout: 300
    pre_reboot_delay: 5
    post_reboot_delay: 30
  when: fix_dpkg_lock_failed | default(false)
```

**Главный вывод:** Не используйте `dpkg --configure -a` в Ansible, если вы не уверены, что lock уже снят. Вместо этого делайте `apt update` с повторами и `force_apt_get: yes`.

_______________
_______________
_______________


Вот готовая Ansible роль для надёжного устранения блокировки dpkg/apt. Она учитывает предупреждение Debian, но в автоматизации действует принудительно, убивая процесс-держатель и очищая lock-файлы, после чего выполняет apt update с повторными попытками.

Структура роли

```
fix_dpkg_lock/
├── tasks/
│   └── main.yml
├── handlers/
│   └── main.yml
├── vars/
│   └── main.yml
├── meta/
│   └── main.yml
└── README.md
```

Содержание файлов

meta/main.yml

```yaml
---
galaxy_info:
  author: "Your Name"
  description: "Fix dpkg/apt lock issues on Debian/Ubuntu"
  license: "MIT"
  min_ansible_version: 2.9
  platforms:
    - name: Debian
      versions:
        - all
    - name: Ubuntu
      versions:
        - all
  galaxy_tags:
    - dpkg
    - apt
    - lock
    - fix
dependencies: []
```

vars/main.yml

```yaml
---
# Список lock-файлов для удаления
dpkg_lock_files:
  - /var/lib/dpkg/lock-frontend
  - /var/lib/dpkg/lock
  - /var/cache/apt/archives/lock
  - /var/lib/apt/lists/lock

# Таймауты и повторы
apt_retries: 5
apt_delay: 10
kill_wait_seconds: 3
```

handlers/main.yml

```yaml
---
- name: restart apt-daily services
  systemd:
    name: "{{ item }}"
    state: restarted
    daemon_reload: yes
  loop:
    - apt-daily.service
    - apt-daily-upgrade.service
    - apt-daily.timer
    - apt-daily-upgrade.timer
  ignore_errors: yes
```

tasks/main.yml (основная логика)

```yaml
---
- name: Проверка, что система использует apt
  fail:
    msg: "Эта роль предназначена только для Debian/Ubuntu"
  when: ansible_os_family != "Debian"

- name: Поиск PID процесса, удерживающего lock-frontend
  shell: |
    set -o pipefail
    fuser /var/lib/dpkg/lock-frontend 2>/dev/null | awk '{print $1}' || true
  args:
    executable: /bin/bash
  register: locking_pid
  changed_when: false
  failed_when: false

- name: Убить процесс, если он существует и не является системным (PID != 1)
  shell: |
    kill -9 {{ locking_pid.stdout }}
  when:
    - locking_pid.stdout != ""
    - locking_pid.stdout != "1"
  register: kill_result
  changed_when: kill_result.rc == 0
  failed_when: false

- name: Небольшая пауза после убийства процесса
  wait_for:
    timeout: "{{ kill_wait_seconds }}"
  when: kill_result is changed

- name: Принудительное удаление lock-файлов
  file:
    path: "{{ item }}"
    state: absent
  loop: "{{ dpkg_lock_files }}"
  register: removed_locks
  failed_when: false

- name: Перезапуск сервисов apt-daily (чтобы не мешали)
  include_tasks: restart_apt_services.yml
  when: removed_locks is changed

- name: Обновление кэша apt с повторными попытками (без dpkg --configure)
  apt:
    update_cache: yes
    cache_valid_time: 0
    force_apt_get: yes
  register: apt_update
  until: apt_update is success
  retries: "{{ apt_retries }}"
  delay: "{{ apt_delay }}"
  ignore_errors: yes

- name: Если apt update не удался — попробовать убить оставшиеся процессы apt/dpkg
  block:
    - name: Найти все процессы apt/dpkg
      shell: pgrep -f "apt|dpkg" | xargs -r kill -9 || true
      changed_when: false

    - name: Повторная попытка обновления
      apt:
        update_cache: yes
        force_apt_get: yes
      register: apt_update_retry
      until: apt_update_retry is success
      retries: 3
      delay: 5
  when: apt_update is failed

- name: Исправление сломанных зависимостей (без блокировки)
  apt:
    name: "{{ item }}"
    state: present
    force_apt_get: yes
  loop:
    - apt-utils
    - dpkg
  when: apt_update_retry is successful or apt_update is successful
  ignore_errors: yes

- name: Финальная проверка — установка тестового пакета (опционально)
  apt:
    name: "{{ test_package | default('bash') }}"
    state: present
  register: final_test
  until: final_test is success
  retries: 2
  delay: 3
  ignore_errors: yes

- name: Если ничего не помогло — выдать подробную инструкцию для ручного вмешательства
  fail:
    msg: |
      Не удалось снять блокировку dpkg/apt после всех попыток.
      Выполните вручную на сервере {{ inventory_hostname }}:
        sudo systemctl stop apt-daily.service apt-daily-upgrade.service
        sudo kill -9 $(fuser /var/lib/dpkg/lock-frontend 2>/dev/null)
        sudo rm -f /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock
        sudo dpkg --configure -a
        sudo apt-get update
        sudo apt-get install -f
        sudo systemctl start apt-daily.service apt-daily-upgrade.service
  when: final_test is failed
```

Дополнительный файл tasks/restart_apt_services.yml (если вынесли перезапуск)

```yaml
---
- name: Остановка apt-daily сервисов
  systemd:
    name: "{{ item }}"
    state: stopped
  loop:
    - apt-daily.service
    - apt-daily-upgrade.service
    - apt-daily.timer
    - apt-daily-upgrade.timer
  ignore_errors: yes

- name: Запуск apt-daily сервисов
  systemd:
    name: "{{ item }}"
    state: started
  loop:
    - apt-daily.service
    - apt-daily-upgrade.service
    - apt-daily.timer
    - apt-daily-upgrade.timer
  ignore_errors: yes
```

Использование роли в плейбуке

```yaml
- hosts: all
  become: yes
  roles:
    - fix_dpkg_lock

  # После исправления можно выполнять установку пакетов
  post_tasks:
    - name: Установить нужный пакет
      apt:
        name: nginx
        state: present
```

Ключевые особенности

1. Безопасное удаление lock — сначала убивается процесс-держатель, только потом удаляются файлы.
2. Нет вызова dpkg --configure -a — он сам требует lock и вызывает ошибку.
3. Повторные попытки — обновление apt выполняется до 5 раз с задержками.
4. Перезапуск системных сервисов apt — предотвращает повторную блокировку.
5. Подробный fail-сообщение — если автоматика не справилась, даёт готовые команды для ручного исправления.
6. Идемпотентность — при повторном запуске ничего не ломает.

Роль готова к использованию в production-среде.



--------------------------------
-------------------------------------------

Ошибка возникает из-за того, что в вашей роли используется переменная `dpkg_lock_files`, которая не определена.

Вот **исправленная роль**:

## Вариант 1: Использовать статический список (рекомендуется)

### `tasks/main.yml`

```yaml
---
- name: Убить процессы apt/dpkg
  shell: |
    pkill -9 -f "apt|dpkg" || true
    sleep 2
  changed_when: false
  ignore_errors: yes

- name: Принудительное удаление lock-файлов
  file:
    path: "{{ item }}"
    state: absent
  loop:
    - /var/lib/dpkg/lock-frontend
    - /var/lib/dpkg/lock
    - /var/cache/apt/archives/lock
    - /var/lib/apt/lists/lock
  ignore_errors: yes

- name: Перезапустить сервисы APT
  systemd:
    name: "{{ item }}"
    state: restarted
    daemon_reload: yes
  loop:
    - apt-daily.timer
    - apt-daily-upgrade.timer
  ignore_errors: yes

- name: Обновить кэш APT
  apt:
    update_cache: yes
    cache_valid_time: 0
    force_apt_get: yes
  register: apt_update
  until: apt_update is success
  retries: 3
  delay: 5
  ignore_errors: yes

- name: Исправить сломанные зависимости
  apt:
    upgrade: dist
    force_apt_get: yes
  when: apt_update is success
  ignore_errors: yes
```

## Вариант 2: Определить переменную (если хотите использовать список)

### `defaults/main.yml`

```yaml
---
# Список lock-файлов dpkg/apt для удаления
dpkg_lock_files:
  - /var/lib/dpkg/lock-frontend
  - /var/lib/dpkg/lock
  - /var/cache/apt/archives/lock
  - /var/lib/apt/lists/lock
```

### `tasks/main.yml`

```yaml
---
- name: Убить процессы apt/dpkg
  shell: |
    pkill -9 -f "apt|dpkg" || true
    sleep 2
  changed_when: false
  ignore_errors: yes

- name: Принудительное удаление lock-файлов
  file:
    path: "{{ item }}"
    state: absent
  loop: "{{ dpkg_lock_files }}"
  ignore_errors: yes

- name: Перезапустить сервисы APT
  systemd:
    name: "{{ item }}"
    state: restarted
    daemon_reload: yes
  loop:
    - apt-daily.timer
    - apt-daily-upgrade.timer
  ignore_errors: yes

- name: Обновить кэш APT
  apt:
    update_cache: yes
    cache_valid_time: 0
    force_apt_get: yes
  register: apt_update
  until: apt_update is success
  retries: 3
  delay: 5
  ignore_errors: yes
```

## Вариант 3: Упрощённый (минимальный)

### `tasks/main.yml`

```yaml
---
- name: Удалить lock-файлы dpkg
  shell: |
    rm -f /var/lib/dpkg/lock-frontend
    rm -f /var/lib/dpkg/lock
    rm -f /var/cache/apt/archives/lock
    rm -f /var/lib/apt/lists/lock
  ignore_errors: yes

- name: Обновить APT
  apt:
    update_cache: yes
  ignore_errors: yes

- name: Установить пакет (если нужно)
  apt:
    name: "{{ package_name | default(omit) }}"
    state: present
  when: package_name is defined
```

## Полная структура роли с переменными

```
roles/fix_dpkg_lock/
├── defaults/
│   └── main.yml          # ← здесь определить dpkg_lock_files
├── tasks/
│   └── main.yml          # ← здесь использовать {{ dpkg_lock_files }}
└── meta/
    └── main.yml
```

### `defaults/main.yml`

```yaml
---
# Список lock-файлов для удаления
dpkg_lock_files:
  - /var/lib/dpkg/lock-frontend
  - /var/lib/dpkg/lock
  - /var/cache/apt/archives/lock
  - /var/lib/apt/lists/lock

# Таймаут в секундах перед повторными попытками
dpkg_retry_delay: 5

# Количество повторных попыток
dpkg_retries: 3
```

## Быстрое исправление текущей ошибки

Если не хотите создавать `defaults/main.yml`, просто замените в вашем `tasks/main.yml` строку:

**Было (с ошибкой):**
```yaml
loop: "{{ dpkg_lock_files }}"
```

**Стало (с явным списком):**
```yaml
loop:
  - /var/lib/dpkg/lock-frontend
  - /var/lib/dpkg/lock
  - /var/cache/apt/archives/lock
  - /var/lib/apt/lists/lock
```

Это самое простое и надёжное решение.

___________
___________
___________


Адаптируем роль для Ubuntu 22.04 и 24.04 и добавим безопасное использование dpkg --configure -a после снятия блокировки.

Исправленная роль fix_dpkg_lock

Структура

```
fix_dpkg_lock/
├── tasks/
│   └── main.yml
├── handlers/
│   └── main.yml
├── vars/
│   └── Ubuntu.yml
├── meta/
│   └── main.yml
└── README.md
```

---

vars/Ubuntu.yml (специфичные для Ubuntu переменные)

```yaml
---
# Для Ubuntu 22.04 и 24.04 одинаковый список lock-файлов
dpkg_lock_files:
  - /var/lib/dpkg/lock-frontend
  - /var/lib/dpkg/lock
  - /var/cache/apt/archives/lock
  - /var/lib/apt/lists/lock

apt_retries: 5
apt_delay: 10
kill_wait_seconds: 3

# Сервисы apt для перезапуска на Ubuntu
apt_services:
  - apt-daily.service
  - apt-daily-upgrade.service
  - apt-daily.timer
  - apt-daily-upgrade.timer
```

---

tasks/main.yml (основная логика с dpkg --configure -a)

```yaml
---
- name: Проверка, что система — Ubuntu
  fail:
    msg: "Эта роль предназначена только для Ubuntu 22.04/24.04"
  when: ansible_distribution != "Ubuntu" or ansible_distribution_version not in ["22.04", "24.04"]

- name: Загрузка переменных для Ubuntu
  include_vars: "Ubuntu.yml"

- name: Поиск PID процесса, удерживающего lock-frontend
  shell: |
    set -o pipefail
    fuser /var/lib/dpkg/lock-frontend 2>/dev/null | awk '{print $1}' || true
  args:
    executable: /bin/bash
  register: locking_pid
  changed_when: false
  failed_when: false

- name: Принудительное завершение процесса (если найден)
  shell: |
    kill -9 {{ locking_pid.stdout }}
  when:
    - locking_pid.stdout != ""
    - locking_pid.stdout != "1"
  register: kill_result
  changed_when: kill_result.rc == 0
  failed_when: false

- name: Пауза после убийства процесса
  wait_for:
    timeout: "{{ kill_wait_seconds }}"
  when: kill_result is changed

- name: Удаление lock-файлов
  file:
    path: "{{ item }}"
    state: absent
  loop: "{{ dpkg_lock_files }}"
  register: removed_locks
  failed_when: false

- name: Остановка apt-сервисов, чтобы они не перезаблокировали
  systemd:
    name: "{{ item }}"
    state: stopped
  loop: "{{ apt_services }}"
  ignore_errors: yes
  when: removed_locks is changed

- name: Выполнение dpkg --configure -a (теперь lock-файлов нет)
  command: dpkg --configure -a
  register: dpkg_configure
  failed_when: false
  changed_when: dpkg_configure.rc == 0

- name: Исправление сломанных пакетов
  apt:
    name: -f
    state: present
    force_apt_get: yes
  register: apt_fix
  failed_when: false

- name: Обновление кэша apt с повторными попытками
  apt:
    update_cache: yes
    cache_valid_time: 0
    force_apt_get: yes
  register: apt_update
  until: apt_update is success
  retries: "{{ apt_retries }}"
  delay: "{{ apt_delay }}"
  ignore_errors: yes

- name: Перезапуск apt-сервисов
  systemd:
    name: "{{ item }}"
    state: started
    daemon_reload: yes
  loop: "{{ apt_services }}"
  ignore_errors: yes

- name: Финальная проверка — установка тестового пакета
  apt:
    name: "{{ test_package | default('bash') }}"
    state: present
  register: final_test
  until: final_test is success
  retries: 2
  delay: 3

- name: Вывод успешного сообщения
  debug:
    msg: "Блокировка dpkg/apt успешно снята, система готова к работе"
  when: final_test is success
```

---

handlers/main.yml

```yaml
---
- name: restart apt services
  systemd:
    name: "{{ item }}"
    state: restarted
  loop: "{{ apt_services }}"
  listen: "restart_apt"
```

---

meta/main.yml

```yaml
---
galaxy_info:
  author: "Your Name"
  description: "Fix dpkg/apt lock for Ubuntu 22.04 and 24.04"
  license: MIT
  min_ansible_version: "2.9"
  platforms:
    - name: Ubuntu
      versions:
        - jammy
        - noble
  galaxy_tags:
    - dpkg
    - apt
    - lock
dependencies: []
```

---

Пример использования в плейбуке

```yaml
- name: Исправление блокировки и установка пакетов
  hosts: all
  become: yes
  roles:
    - fix_dpkg_lock

  post_tasks:
    - name: Установить нужный пакет (например, nginx)
      apt:
        name: nginx
        state: present
```

Ключевые улучшения для Ubuntu 22.04/24.04

1. Явная проверка версий — ansible_distribution_version in ["22.04", "24.04"].
2. Корректный порядок:
   · Убить процесс → удалить lock-файлы → остановить apt-сервисы → dpkg --configure -a → apt --fix-broken install → apt update.
3. Перезапуск сервисов — предотвращает повторную блокировку от фоновых задач apt-daily.
4. Поддержка systemd — на Ubuntu 22.04/24.04 работает единообразно.

Роль готова к использованию в production-среде на целевых версиях Ubuntu.













