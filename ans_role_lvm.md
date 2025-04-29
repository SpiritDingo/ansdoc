# Ansible роль для управления LVM на Linux серверах

Ниже представлена структура Ansible роли для управления Logical Volume Manager (LVM) на Linux серверах, включая создание, изменение и удаление физических томов (PV), групп томов (VG) и логических томов (LV).

## Структура роли

```
roles/lvm_management/
├── defaults
│   └── main.yml       # Основные переменные по умолчанию
├── tasks
│   ├── main.yml       # Основные задачи
│   ├── pv.yml         # Управление физическими томами
│   ├── vg.yml         # Управление группами томов
│   ├── lv.yml         # Управление логическими томами
│   └── fs.yml         # Управление файловыми системами
├── handlers
│   └── main.yml       # Обработчики для перезагрузки сервисов
└── vars
    └── main.yml       # Специфичные переменные
```

## Файлы роли

### defaults/main.yml

```yaml
---
# Настройки по умолчанию для LVM
lvm_management:
  pvs: []              # Список физических томов для управления
  vgs: []              # Список групп томов для управления
  lvs: []              # Список логических томов для управления
  fs_types:            # Поддерживаемые типы файловых систем
    - ext4
    - xfs
    - ext3
    - btrfs
  default_fs_type: ext4
  mount_options: defaults
```

### tasks/main.yml

```yaml
---
- name: Install LVM2 package
  ansible.builtin.package:
    name: lvm2
    state: present

- name: Include physical volumes tasks
  ansible.builtin.include_tasks: pv.yml
  when: lvm_management.pvs | length > 0

- name: Include volume groups tasks
  ansible.builtin.include_tasks: vg.yml
  when: lvm_management.vgs | length > 0

- name: Include logical volumes tasks
  ansible.builtin.include_tasks: lv.yml
  when: lvm_management.lvs | length > 0

- name: Include filesystem tasks
  ansible.builtin.include_tasks: fs.yml
  when: lvm_management.lvs | length > 0
```

### tasks/pv.yml

```yaml
---
- name: Check existing physical volumes
  ansible.builtin.command: pvs --noheadings -o pv_name
  register: existing_pvs
  changed_when: false
  tags: lvm

- name: Create physical volumes
  ansible.builtin.command: pvcreate {{ item.device }}
  with_items: "{{ lvm_management.pvs }}"
  when: item.device not in existing_pvs.stdout
  tags: lvm

- name: Remove physical volumes (if state=absent)
  ansible.builtin.command: pvremove {{ item.device }}
  with_items: "{{ lvm_management.pvs }}"
  when: item.state is defined and item.state == 'absent' and item.device in existing_pvs.stdout
  tags: lvm
```

### tasks/vg.yml

```yaml
---
- name: Check existing volume groups
  ansible.builtin.command: vgs --noheadings -o vg_name
  register: existing_vgs
  changed_when: false
  tags: lvm

- name: Create volume groups
  ansible.builtin.command: >
    vgcreate {{ item.name }}
    {% if item.pvs is defined %}{{ item.pvs | join(' ') }}{% endif %}
    {% if item.options is defined %}{{ item.options }}{% endif %}
  with_items: "{{ lvm_management.vgs }}"
  when: item.name not in existing_vgs.stdout and item.state is not defined or item.state != 'absent'
  tags: lvm

- name: Extend volume groups
  ansible.builtin.command: >
    vgextend {{ item.name }} {{ item.pvs | join(' ') }}
  with_items: "{{ lvm_management.vgs }}"
  when: 
    - item.name in existing_vgs.stdout
    - item.pvs is defined
    - item.state is not defined or item.state != 'absent'
  tags: lvm

- name: Remove volume groups (if state=absent)
  ansible.builtin.command: vgremove {{ item.name }}
  with_items: "{{ lvm_management.vgs }}"
  when: item.state is defined and item.state == 'absent' and item.name in existing_vgs.stdout
  tags: lvm
```

### tasks/lv.yml

```yaml
---
- name: Check existing logical volumes
  ansible.builtin.command: lvs --noheadings -o lv_name,vg_name
  register: existing_lvs
  changed_when: false
  tags: lvm

- name: Create logical volumes
  ansible.builtin.command: >
    lvcreate -n {{ item.name }} 
    {% if item.size is defined %}-L {{ item.size }}{% endif %}
    {% if item.extents is defined %}-l {{ item.extents }}{% endif %}
    {% if item.vg is defined %}{{ item.vg }}{% endif %}
    {% if item.options is defined %}{{ item.options }}{% endif %}
  with_items: "{{ lvm_management.lvs }}"
  when: >
    (item.vg + "/" + item.name) not in existing_lvs.stdout and
    (item.state is not defined or item.state != 'absent')
  tags: lvm

- name: Resize logical volumes
  block:
    - name: Check current LV size
      ansible.builtin.command: lvs --noheadings -o lv_size {{ item.vg }}/{{ item.name }}
      register: current_lv_size
      changed_when: false

    - name: Resize LV
      ansible.builtin.command: >
        lvresize -L {{ item.size }} {{ item.vg }}/{{ item.name }}
      when: current_lv_size.stdout | regex_replace('[^0-9.]', '') | float < item.size | regex_replace('[^0-9.]', '') | float
  with_items: "{{ lvm_management.lvs }}"
  when: >
    item.size is defined and
    (item.vg + "/" + item.name) in existing_lvs.stdout and
    (item.state is not defined or item.state != 'absent')
  tags: lvm

- name: Remove logical volumes (if state=absent)
  ansible.builtin.command: lvremove -f {{ item.vg }}/{{ item.name }}
  with_items: "{{ lvm_management.lvs }}"
  when: >
    item.state is defined and item.state == 'absent' and
    (item.vg + "/" + item.name) in existing_lvs.stdout
  tags: lvm
```

### tasks/fs.yml

```yaml
---
- name: Create filesystems on logical volumes
  ansible.builtin.filesystem:
    fstype: "{{ item.fs_type | default(lvm_management.default_fs_type) }}"
    dev: "/dev/{{ item.vg }}/{{ item.name }}"
  with_items: "{{ lvm_management.lvs }}"
  when: >
    (item.state is not defined or item.state != 'absent') and
    (item.mount is defined or item.fs_type is defined)
  tags: lvm

- name: Mount logical volumes
  ansible.builtin.mount:
    path: "{{ item.mount }}"
    src: "/dev/{{ item.vg }}/{{ item.name }}"
    fstype: "{{ item.fs_type | default(lvm_management.default_fs_type) }}"
    state: "{{ 'mounted' if (item.state is not defined or item.state != 'absent') else 'absent' }}"
    opts: "{{ item.mount_options | default(lvm_management.mount_options) }}"
  with_items: "{{ lvm_management.lvs }}"
  when: item.mount is defined
  tags: lvm
```

### handlers/main.yml

```yaml
---
- name: Reload systemd mounts
  ansible.builtin.systemd:
    daemon_reload: yes

- name: Remount all
  ansible.builtin.command: mount -a
  when: ansible_facts['os_family'] != 'RedHat'
```

## Пример использования роли

1. Создайте playbook `manage_lvm.yml`:

```yaml
---
- hosts: storage_servers
  become: yes
  roles:
    - lvm_management
  vars:
    lvm_management:
      pvs:
        - device: /dev/sdb
        - device: /dev/sdc
          state: absent  # Удалить этот PV если существует

      vgs:
        - name: vg_data
          pvs:
            - /dev/sdb
          options: "--physicalextentsize 16M"

      lvs:
        - name: lv_app
          vg: vg_data
          size: 20G
          fs_type: xfs
          mount: /opt/app
          mount_options: "defaults,noatime"

        - name: lv_logs
          vg: vg_data
          size: 10G
          fs_type: ext4
          mount: /var/log
```

2. Запустите playbook:

```bash
ansible-playbook -i inventory.ini manage_lvm.yml
```

## Дополнительные возможности

1. **Расширенное управление размерами**:
   - Поддержка относительных размеров (+100%FREE)
   - Тонкая настройка thin provisioning

2. **Резервное копирование метаданных LVM**:
   - Автоматическое сохранение конфигурации VG/LV
   - Восстановление из резервной копии

3. **Мониторинг LVM**:
   - Проверка свободного места
   - Оповещения при достижении пороговых значений

4. **Оптимизация производительности**:
   - Настройка striping для повышения производительности
   - Конфигурация зеркалирования для отказоустойчивости

5. **Миграция данных**:
   - Перемещение LV между PV
   - Изменение размера PV без простоев

Эта роль предоставляет комплексное решение для управления LVM на Linux серверах с поддержкой всех основных операций: от создания физических томов до монтирования файловых систем.