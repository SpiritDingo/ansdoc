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
