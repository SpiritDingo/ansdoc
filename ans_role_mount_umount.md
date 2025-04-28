# Ansible роль для монтирования и отмонтирования сетевых папок с настройкой fstab

Роль позволяет управлять сетевыми файловыми системами (NFS, CIFS/Samba) с автоматической настройкой `/etc/fstab`.

## Структура роли

```
roles/network_mount/
├── defaults
│   └── main.yml
├── tasks
│   ├── main.yml
│   ├── mount.yml
│   ├── umount.yml
│   └── fstab.yml
├── templates
│   └── fstab.j2
└── README.md
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Список сетевых монтирований
network_mounts:
  - name: "nfs_share"               # Уникальное имя монтирования
    state: "present"                # present/absent - монтировать или удалить
    mount_point: "/mnt/nfs_share"   # Локальная точка монтирования
    src: "nfs-server:/export/share" # Источник (сервер:путь)
    fstype: "nfs"                   # Тип ФС (nfs, cifs, smbfs)
    opts: "rw,noatime,soft"        # Опции монтирования
    dump: "0"                       # Параметр dump для fstab
    passno: "0"                     # Параметр passno для fstab
    create_mount_point: true        # Создавать ли точку монтирования
    mount_point_owner: "root"       # Владелец точки монтирования
    mount_point_group: "root"       # Группа точки монтирования
    mount_point_mode: "0755"        # Права на точку монтирования

# Параметры для аутентификации (для CIFS/Samba)
cifs_username: ""
cifs_password: ""
cifs_domain: ""
credentials_file: "/etc/.smbcredentials"  # Файл с учетными данными для CIFS
```

### tasks/main.yml

```yaml
---
- name: Include mount tasks
  include_tasks: mount.yml
  when: item.state == 'present'
  loop: "{{ network_mounts }}"
  loop_control:
    loop_var: mount_item

- name: Include umount tasks
  include_tasks: umount.yml
  when: item.state == 'absent'
  loop: "{{ network_mounts }}"
  loop_control:
    loop_var: mount_item

- name: Update fstab configuration
  include_tasks: fstab.yml
```

### tasks/mount.yml

```yaml
---
- name: Install required packages
  package:
    name:
      - "nfs-common"  # Для NFS
      - "cifs-utils"  # Для CIFS/Samba
    state: present
  when: mount_item.fstype in ['nfs', 'cifs', 'smbfs']

- name: Create credentials file for CIFS
  template:
    src: "credentials.j2"
    dest: "{{ credentials_file }}"
    owner: root
    group: root
    mode: '0600'
  when:
    - mount_item.fstype in ['cifs', 'smbfs']
    - cifs_username | length > 0

- name: Create mount point directory
  file:
    path: "{{ mount_item.mount_point }}"
    state: directory
    owner: "{{ mount_item.mount_point_owner }}"
    group: "{{ mount_item.mount_point_group }}"
    mode: "{{ mount_item.mount_point_mode }}"
  when: mount_item.create_mount_point | bool

- name: Mount network share (temporary)
  mount:
    path: "{{ mount_item.mount_point }}"
    src: "{{ mount_item.src }}"
    fstype: "{{ mount_item.fstype }}"
    opts: "{{ mount_item.opts }}"
    state: mounted
```

### tasks/umount.yml

```yaml
---
- name: Unmount network share
  mount:
    path: "{{ mount_item.mount_point }}"
    src: "{{ mount_item.src }}"
    fstype: "{{ mount_item.fstype }}"
    state: unmounted
  ignore_errors: yes

- name: Remove mount point directory
  file:
    path: "{{ mount_item.mount_point }}"
    state: absent
  when: mount_item.create_mount_point | bool
```

### tasks/fstab.yml

```yaml
---
- name: Generate fstab entries
  template:
    src: fstab.j2
    dest: /etc/fstab
    owner: root
    group: root
    mode: '0644'
```

### templates/fstab.j2

```
# Ansible managed - do not edit manually

# Static entries
{{ ansible_managed | comment }}

# Existing entries (preserved)
{% for line in fstab_content.split('\n') if not line.startswith('#') and line.strip() != '' and line not in ansible_managed %}
{{ line }}
{% endfor %}

# Network mounts
{% for item in network_mounts if item.state == 'present' %}
{% if item.fstype in ['cifs', 'smbfs'] and cifs_username | length > 0 %}
{{ item.src }} {{ item.mount_point }} {{ item.fstype }} {{ item.opts }},credentials={{ credentials_file }} {{ item.dump }} {{ item.passno }}
{% else %}
{{ item.src }} {{ item.mount_point }} {{ item.fstype }} {{ item.opts }} {{ item.dump }} {{ item.passno }}
{% endif %}
{% endfor %}
```

## Пример использования

1. Создайте playbook `setup_network_mounts.yml`:

```yaml
---
- hosts: all
  become: yes
  roles:
    - network_mount
  vars:
    network_mounts:
      - name: "nfs_data"
        state: "present"
        mount_point: "/mnt/data"
        src: "nas01:/export/data"
        fstype: "nfs"
        opts: "rw,noatime,soft,timeo=180"
        dump: "0"
        passno: "0"
        create_mount_point: true

      - name: "samba_share"
        state: "present"
        mount_point: "/mnt/samba"
        src: "//fileserver/share"
        fstype: "cifs"
        opts: "rw,uid=1000,gid=1000,file_mode=0660,dir_mode=0770"
        dump: "0"
        passno: "0"
        create_mount_point: true

    cifs_username: "user"
    cifs_password: "password"
    cifs_domain: "WORKGROUP"
```

2. Запустите playbook:
```bash
ansible-playbook -i inventory setup_network_mounts.yml
```

3. Для удаления монтирования измените `state: "absent"` и снова запустите playbook.

## Особенности

- Поддержка NFS и CIFS/Samba
- Автоматическое создание/удаление точек монтирования
- Безопасное хранение учетных данных для CIFS
- Сохранение существующих записей в fstab
- Возможность временного монтирования без изменения fstab
- Поддержка различных опций монтирования

Для работы с другими типами сетевых файловых систем (например, SSHFS) можно расширить роль, добавив соответствующие задачи и пакеты.