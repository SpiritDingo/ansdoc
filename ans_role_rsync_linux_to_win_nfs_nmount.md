# Ansible роль для копирования файлов с Linux на Windows NFS без монтирования

Вот готовая Ansible роль для копирования файлов с Linux сервера на Windows NFS общий ресурс без монтирования, используя прямое подключение через rsync и cron.

## Структура роли

```
roles/rsync_to_win_nfs_direct/
├── tasks/
│   └── main.yml
├── defaults/
│   └── main.yml
├── templates/
│   └── rsync_cron.j2
```

## 1. Файл defaults/main.yml

```yaml
---
# Исходный каталог на Linux сервере
rsync_src: "/path/to/source"

# Адрес Windows NFS сервера и общий ресурс
nfs_server: "windows-nfs.example.com"
nfs_share: "/backup_share"

# Параметры rsync для работы с Windows NFS
rsync_options: "-rtv --chmod=ugo=rwX --no-perms --no-owner --no-group --progress"

# Пользователь для cron задачи
cron_user: "root"

# Расписание cron (по умолчанию ежедневно в 2:00)
cron_schedule: "0 2 * * *"

# Имя cron задачи
cron_job_name: "RSync to Windows NFS"

# Лог-файл
rsync_log_file: "/var/log/rsync_to_windows_nfs.log"

# Таймаут подключения (секунды)
rsync_timeout: 300
```

## 2. Файл tasks/main.yml

```yaml
---
- name: Install rsync package
  package:
    name: rsync
    state: present

- name: Install NFS client utilities
  package:
    name: nfs-common
    state: present

- name: Create log directory
  file:
    path: "{{ rsync_log_file | dirname }}"
    state: directory
    mode: '0755'

- name: Deploy rsync cron job
  template:
    src: rsync_cron.j2
    dest: "/etc/cron.d/rsync_to_windows_nfs"
    owner: root
    group: root
    mode: '0644'
  tags: cron

- name: Verify NFS connection
  command: >
    /usr/bin/rsync {{ rsync_options }} --timeout={{ rsync_timeout }} --dry-run
    {{ rsync_src }}/ {{ nfs_server }}:{{ nfs_share }}/
  register: nfs_test
  changed_when: false
  ignore_errors: yes
  tags: verify

- name: Show connection test results
  debug:
    msg: "NFS connection test {{ 'successful' if nfs_test.rc == 0 else 'failed' }}"
  tags: verify
```

## 3. Шаблон templates/rsync_cron.j2

```jinja2
# {{ cron_job_name }}
{{ cron_schedule }} {{ cron_user }} /usr/bin/rsync {{ rsync_options }} --timeout={{ rsync_timeout }} {{ rsync_src }}/ {{ nfs_server }}:{{ nfs_share }}/ >> {{ rsync_log_file }} 2>&1
```

## 4. Пример использования в playbook

```yaml
---
- hosts: linux_servers
  become: yes
  roles:
    - rsync_to_win_nfs_direct
  vars:
    rsync_src: "/data/important_files"
    nfs_server: "backup-win.example.com"
    nfs_share: "/backups"
    cron_schedule: "30 3 * * *"  # Каждый день в 3:30 утра
    rsync_options: "-azv --chmod=ugo=rwX --no-perms --no-owner --no-group --delete"
```

## Особенности реализации

1. **Прямое подключение к NFS без монтирования**:
   - Используется синтаксис `server:/share` в rsync
   - Не требует постоянного монтированного ресурса

2. **Оптимизированные параметры rsync**:
   - `--no-perms --no-owner --no-group` - игнорирует права, которые могут вызвать проблемы с Windows
   - `--chmod=ugo=rwX` - устанавливает базовые права
   - `--timeout` - предотвращает зависание при проблемах с сетью

3. **Проверка подключения**:
   - Роль включает тестовую задачу для проверки доступности NFS ресурса
   - Тест выполняется с `--dry-run` для безопасной проверки

## Настройка Windows NFS сервера

1. На Windows сервере:
   - Установите "Сервер NFS" через "Установка ролей и компонентов"
   - Настройте общий ресурс NFS
   - В свойствах общего ресурса:
     - Разрешите анонимный доступ (UID/GID = 0)
     - Установите разрешения на чтение/запись

2. Для тестирования вручную выполните:
```bash
rsync -avn --no-perms --no-owner --no-group /local/path/ windows-nfs-server:/share/
```

## Дополнительные рекомендации

1. Для больших объемов данных рассмотрите:
   ```yaml
   rsync_options: "-azv --chmod=ugo=rwX --no-perms --no-owner --no-group --bwlimit=50000"
   ```

2. Для уведомлений об ошибках добавьте в cron:
   ```jinja2
   {{ cron_schedule }} {{ cron_user }} /usr/bin/rsync {{ rsync_options }} {{ rsync_src }}/ {{ nfs_server }}:{{ nfs_share }}/ >> {{ rsync_log_file }} 2>&1 || echo "RSync failed $(date)" | mail -s "RSync error" admin@example.com
   ```

3. Для ротации логов добавьте файл `/etc/logrotate.d/rsync_to_windows_nfs`:
   ```
   {{ rsync_log_file }} {
       weekly
       rotate 4
       compress
       missingok
       notifempty
   }
   ```