# Ansible роль для копирования файлов на Windows NFS с аутентификацией

Для копирования файлов с Linux на Windows NFS сервер с использованием учётных данных (без монтирования) через cron, вот полное решение:

## Структура роли

```
roles/rsync_win_nfs_auth/
├── tasks/
│   ├── main.yml
├── defaults/
│   ├── main.yml
├── templates/
│   ├── rsync_cron.j2
│   ├── rsync_password_file.j2
├── files/
│   ├── nfs_credentials_example
```

## 1. Файл defaults/main.yml

```yaml
---
# Исходный каталог для копирования
rsync_src: "/path/to/source"

# Параметры Windows NFS сервера
nfs_server: "windows-nfs.example.com"
nfs_share: "/backup_share"
nfs_username: "nfs_user"
nfs_password: "secure_password"  # В реальном использовании используйте ansible-vault

# Параметры rsync
rsync_options: "-rtv --progress --no-perms --no-owner --no-group --chmod=ugo=rwX"

# Настройки безопасности
password_file: "/etc/rsync_nfs_password"  # Файл с паролем
password_file_mode: "0600"

# Настройки cron
cron_user: "root"
cron_schedule: "0 2 * * *"
cron_job_name: "RSync to Windows NFS with auth"
rsync_log_file: "/var/log/rsync_to_windows_nfs.log"
rsync_timeout: 300
```

## 2. Файл tasks/main.yml

```yaml
---
- name: Install required packages
  package:
    name: "{{ item }}"
    state: present
  loop:
    - rsync
    - nfs-common
    - cifs-utils  # Для некоторых вариантов аутентификации

- name: Create password file for NFS authentication
  template:
    src: rsync_password_file.j2
    dest: "{{ password_file }}"
    owner: root
    group: root
    mode: "{{ password_file_mode }}"
  no_log: true  # Важно для безопасности!

- name: Create log directory
  file:
    path: "{{ rsync_log_file | dirname }}"
    state: directory
    mode: '0755'

- name: Deploy rsync cron job
  template:
    src: rsync_cron.j2
    dest: "/etc/cron.d/rsync_to_windows_nfs_auth"
    owner: root
    group: root
    mode: '0644'

- name: Verify NFS connection
  command: >
    /usr/bin/rsync {{ rsync_options }} --timeout={{ rsync_timeout }} --dry-run
    --password-file={{ password_file }} {{ rsync_src }}/ {{ nfs_username }}@{{ nfs_server }}:{{ nfs_share }}/
  register: nfs_test
  changed_when: false
  ignore_errors: yes
  no_log: true
  tags: verify

- name: Show connection test results
  debug:
    msg: "NFS connection test {{ 'successful' if nfs_test.rc == 0 else 'failed' }}"
  tags: verify
```

## 3. Шаблон templates/rsync_password_file.j2

```jinja2
{{ nfs_password }}
```

## 4. Шаблон templates/rsync_cron.j2

```jinja2
# {{ cron_job_name }}
{{ cron_schedule }} {{ cron_user }} /usr/bin/rsync {{ rsync_options }} --timeout={{ rsync_timeout }} --password-file={{ password_file }} {{ rsync_src }}/ {{ nfs_username }}@{{ nfs_server }}:{{ nfs_share }}/ >> {{ rsync_log_file }} 2>&1
```

## 5. Пример playbook с использованием ansible-vault

```yaml
---
- hosts: linux_servers
  become: yes
  vars_files:
    - vars/nfs_secrets.yml  # Файл, зашифрованный ansible-vault
  roles:
    - rsync_win_nfs_auth
  vars:
    rsync_src: "/data/important_files"
    nfs_server: "backup-win.example.com"
    nfs_share: "/department_backups"
    cron_schedule: "30 3 * * *"
```

## Особенности реализации

1. **Аутентификация**:
   - Используется файл с паролем (`--password-file`)
   - Файл пароля имеет строгие права доступа (0600)
   - Пароль не логируется благодаря `no_log: true`

2. **Безопасность**:
   - Все секреты должны храниться в ansible-vault
   - Пример файла `vars/nfs_secrets.yml` (зашифровать с помощью `ansible-vault encrypt`):
     ```yaml
     nfs_username: "backup_user"
     nfs_password: "s3cr3tP@ssw0rd"
     ```

3. **Параметры rsync**:
   - Оптимизированы для работы с Windows NFS
   - `--no-perms --no-owner --no-group` - игнорирует неприменимые права Linux
   - `--chmod=ugo=rwX` - устанавливает базовые права

## Настройка Windows NFS сервера

1. На Windows Server:
   - Установите "Сервер NFS" через Server Manager
   - Настройте аутентификацию:
     ```powershell
     # Включить аутентификацию
     Set-NfsMappingStore -EnableLocalMapping $true
     # Добавить пользователя
     New-NfsMapping -UserName "nfs_user" -UnixUserID 0 -UnixGroupID 0
     ```
   - Настройте общий ресурс с требуемыми разрешениями

2. Для проверки подключения вручную:
```bash
echo "s3cr3tP@ssw0rd" > /tmp/nfs_pass
chmod 600 /tmp/nfs_pass
rsync -avn --password-file=/tmp/nfs_pass /local/path/ nfs_user@windows-nfs:/share/
rm /tmp/nfs_pass
```

## Дополнительные рекомендации

1. **Ротация логов**:
   ```yaml
   - name: Configure log rotation
     copy:
       dest: "/etc/logrotate.d/rsync_to_windows_nfs"
       content: |
         {{ rsync_log_file }} {
             weekly
             rotate 4
             compress
             missingok
             notifempty
         }
   ```

2. **Уведомления об ошибках**:
   Модифицируйте cron задачу для отправки email при ошибках:
   ```jinja2
   {{ cron_schedule }} {{ cron_user }} (/usr/bin/rsync {{ rsync_options }} --timeout={{ rsync_timeout }} --password-file={{ password_file }} {{ rsync_src }}/ {{ nfs_username }}@{{ nfs_server }}:{{ nfs_share }}/ || echo "RSync failed at $(date)" | mail -s "RSync Error" admin@example.com) >> {{ rsync_log_file }} 2>&1
   ```

3. **Ограничение полосы пропускания**:
   Добавьте в `rsync_options` параметр `--bwlimit=50000` (KB/s) для ограничения скорости.