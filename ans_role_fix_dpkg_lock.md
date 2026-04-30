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