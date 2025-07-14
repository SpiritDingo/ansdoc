# Универсальная Ansible роль для установки RPM и DEB пакетов

Эта роль позволяет устанавливать пакеты как на Oracle Linux 9 (RPM), так и на Ubuntu 22.04/24.04 (DEB) из файлов, хранящихся в самой роли.

## Структура роли

```
universal_package_installer/
├── defaults/
│   └── main.yml
├── files/
│   ├── ubuntu/
│   │   └── package.deb
│   └── oraclelinux/
│       └── package.rpm
├── tasks/
│   └── main.yml
└── vars/
    └── main.yml
```

## Код роли

### defaults/main.yml

```yaml
---
# Имена пакетов (без расширения)
package_name: "my_package"

# Путь для временного хранения пакетов на целевых хостах
temp_path: "/tmp"

# Удалять пакет после установки (true/false)
remove_package_after_install: true
```

### tasks/main.yml

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution|lower }}_{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution|lower }}.yml"
    - "{{ ansible_os_family|lower }}.yml"
  tags: always

- name: Copy package to target host
  ansible.builtin.copy:
    src: "{{ package_source_path }}"
    dest: "{{ temp_path }}/{{ package_file_name }}"
    mode: '0644'

- name: Install package on RPM-based systems
  ansible.builtin.yum:
    name: "{{ temp_path }}/{{ package_file_name }}"
    state: present
  when: ansible_pkg_mgr == "yum"

- name: Install package on DEB-based systems
  ansible.builtin.apt:
    deb: "{{ temp_path }}/{{ package_file_name }}"
    state: present
  when: ansible_pkg_mgr == "apt"

- name: Remove package file if configured
  ansible.builtin.file:
    path: "{{ temp_path }}/{{ package_file_name }}"
    state: absent
  when: remove_package_after_install
```

### vars/oraclelinux_9.yml

```yaml
---
package_file_name: "{{ package_name }}.rpm"
package_source_path: "oraclelinux/{{ package_file_name }}"
```

### vars/ubuntu.yml

```yaml
---
package_file_name: "{{ package_name }}.deb"
package_source_path: "ubuntu/{{ package_file_name }}"
```

## Использование роли

1. Поместите ваши пакеты в соответствующие подкаталоги:
   - RPM пакеты в `files/oraclelinux/`
   - DEB пакеты в `files/ubuntu/`

2. Добавьте роль в ваш playbook:

```yaml
- hosts: all
  roles:
    - universal_package_installer
```

3. Вы можете переопределить параметры по умолчанию:

```yaml
- hosts: all
  roles:
    - role: universal_package_installer
      vars:
        package_name: "custom_package"
        remove_package_after_install: false
```

## Особенности работы

1. Роль автоматически определяет тип ОС и использует соответствующий менеджер пакетов (yum для Oracle Linux, apt для Ubuntu).
2. Пакеты копируются во временную директорию перед установкой.
3. После установки пакет может быть автоматически удален (настраивается параметром `remove_package_after_install`).
4. Роль поддерживает как Ubuntu 22.04, так и 24.04 через общий файл переменных для Ubuntu.

## Дополнительные улучшения

Для более сложных сценариев вы можете расширить роль, добавив:

1. Проверку контрольных сумм пакетов
2. Установку зависимостей
3. Обработку нескольких пакетов одновременно
4. Кастомные скрипты до/после установки

Эта роль предоставляет базовый функционал для кроссплатформенной установки пакетов из файлов в Ansible.


Вот обновленная версия роли с выводом результатов установки:

```yaml
---
- name: Include OS-specific variables
  include_vars: "{{ item }}"
  with_first_found:
    - "{{ ansible_distribution|lower }}_{{ ansible_distribution_major_version }}.yml"
    - "{{ ansible_distribution|lower }}.yml"
    - "{{ ansible_os_family|lower }}.yml"
  tags: always
  register: vars_include
  ignore_errors: yes

- name: Verify package file exists in role
  ansible.builtin.stat:
    path: "{{ role_path }}/files/{{ package_source_path }}"
  register: package_file_stat

- name: Fail if package file not found
  ansible.builtin.fail:
    msg: "Package file {{ package_file_name }} not found in role files"
  when: not package_file_stat.stat.exists

- name: Copy package to target host
  ansible.builtin.copy:
    src: "{{ package_source_path }}"
    dest: "{{ temp_path }}/{{ package_file_name }}"
    mode: '0644'
  register: package_copy

- name: Display copy result
  ansible.builtin.debug:
    msg: "Package successfully copied to {{ temp_path }}/{{ package_file_name }}"
  when: package_copy.changed

- name: Check if package is already installed (RPM)
  ansible.builtin.command:
    cmd: "rpm -q {{ package_name }}"
  register: rpm_check
  ignore_errors: yes
  when: ansible_pkg_mgr == "yum"

- name: Check if package is already installed (DEB)
  ansible.builtin.command:
    cmd: "dpkg -l {{ package_name }}"
  register: deb_check
  ignore_errors: yes
  when: ansible_pkg_mgr == "apt"

- name: Install package on RPM-based systems
  ansible.builtin.yum:
    name: "{{ temp_path }}/{{ package_file_name }}"
    state: present
  register: package_install
  when: ansible_pkg_mgr == "yum"

- name: Install package on DEB-based systems
  ansible.builtin.apt:
    deb: "{{ temp_path }}/{{ package_file_name }}"
    state: present
  register: package_install
  when: ansible_pkg_mgr == "apt"

- name: Display installation result
  ansible.builtin.debug:
    msg: |
      Package installation {{ package_install.changed|ternary('completed successfully', 'skipped (already installed)') }}
      Package details:
      {% if ansible_pkg_mgr == 'yum' %}
        Name: {{ package_install.rpm_name|default(package_name) }}
        Version: {{ package_install.rpm_version|default('unknown') }}
        Architecture: {{ package_install.rpm_arch|default('unknown') }}
      {% elif ansible_pkg_mgr == 'apt' %}
        Name: {{ package_name }}
        Version: {{ package_install.stdout_lines|select('match', 'Version')|first|default('unknown')|regex_replace('.*Version: (.+)', '\\1') }}
      {% endif %}
  when: "'package_install' in vars"

- name: Verify package installation (RPM)
  ansible.builtin.command:
    cmd: "rpm -q {{ package_name }}"
  register: verify_install_rpm
  changed_when: false
  when: ansible_pkg_mgr == "yum"

- name: Verify package installation (DEB)
  ansible.builtin.command:
    cmd: "dpkg -l {{ package_name }}"
  register: verify_install_deb
  changed_when: false
  when: ansible_pkg_mgr == "apt"

- name: Display final verification
  ansible.builtin.debug:
    msg: |
      Package verification:
      {% if ansible_pkg_mgr == 'yum' %}
        {{ verify_install_rpm.stdout }}
      {% elif ansible_pkg_mgr == 'apt' %}
        {{ verify_install_deb.stdout_lines|select('match', '^ii')|first|default('Package not found') }}
      {% endif %}

- name: Remove package file if configured
  ansible.builtin.file:
    path: "{{ temp_path }}/{{ package_file_name }}"
    state: absent
  register: package_cleanup
  when: remove_package_after_install

- name: Display cleanup result
  ansible.builtin.debug:
    msg: "Temporary package file {{ 'removed' if package_cleanup.changed else 'not removed' }}"
  when: "'package_cleanup' in vars"
```

### Улучшения в новой версии:

1. **Проверка существования пакета в роли**:
   - Добавлена проверка, что файл пакета действительно существует

2. **Проверка уже установленных пакетов**:
   - Проверка наличия пакета перед установкой

3. **Подробный вывод результатов**:
   - Информация о копировании пакета
   - Результат установки (успех/уже установлен)
   - Детали пакета (имя, версия, архитектура)

4. **Верификация установки**:
   - Явная проверка что пакет действительно установлен

5. **Отчет об очистке**:
   - Информация об удалении временного файла пакета

6. **Улучшенная обработка ошибок**:
   - Роль явно сообщит если файл пакета не найден

### Пример вывода при выполнении:

```
TASK [universal_package_installer : Display installation result] ***************
ok: [server1] => {
    "msg": "Package installation completed successfully\nPackage details:\nName: my_package\nVersion: 1.2.3\nArchitecture: x86_64"
}

TASK [universal_package_installer : Display final verification] ***************
ok: [server1] => {
    "msg": "Package verification:\nmy_package-1.2.3-1.el9.x86_64"
}

TASK [universal_package_installer : Display cleanup result] ********************
ok: [server1] => {
    "msg": "Temporary package file removed"
}
```

Эта версия роли дает полную информацию о каждом этапе установки пакета, что упрощает диагностику проблем и подтверждение успешного выполнения.



Ошибка указывает, что Ansible не может найти файл пакета `bz_sensors_agent-nizhpharm-18f168d8.soc.bi.zone-v2.22.1-BZ_SOC_with_EDR-2025.06.02T09-39-2.deb` в директории роли. Давайте решим эту проблему.

## Решение проблемы

1. **Проверьте структуру директорий роли**:
   Убедитесь, что файл пакета находится в правильной директории:
   ```
   roles/
     └── universal_package_installer/
         └── files/
             └── ubuntu/
                 └── bz_sensors_agent-nizhpharm-18f168d8.soc.bi.zone-v2.22.1-BZ_SOC_with_EDR-2025.06.02T09-39-2.deb
   ```

2. **Проверьте переменные роли**:
   Убедитесь, что переменная `package_name` или `package_file_name` правильно указана в:
   - `vars/ubuntu.yml` (для Ubuntu)
   - `group_vars/` или `host_vars/`
   - Параметрах роли при вызове

3. **Временное решение**:
   Если нужно быстро исправить, можно явно указать имя файла при вызове роли:

```yaml
- hosts: lebedev-test-vm
  roles:
    - role: universal_package_installer
      vars:
        package_file_name: "bz_sensors_agent-nizhpharm-18f168d8.soc.bi.zone-v2.22.1-BZ_SOC_with_EDR-2025.06.02T09-39-2.deb"
        package_name: "bz_sensors_agent"  # Имя пакета без версии и расширения
```

## Дополнительные проверки

1. **Проверьте права доступа**:
   Убедитесь, что файл пакета доступен для чтения:
   ```bash
   ls -l roles/universal_package_installer/files/ubuntu/
   ```

2. **Проверьте путь к роли**:
   Убедитесь, что Ansible может найти вашу роль. Путь к роли можно проверить так:
   ```bash
   ansible-config dump | grep DEFAULT_ROLES_PATH
   ```

3. **Альтернативный вариант**:
   Если пакет очень большой или часто меняется, рассмотрите возможность хранения его на файловом сервере и скачивания перед установкой:

```yaml
- name: Download package from remote server
  ansible.builtin.get_url:
    url: "http://your-server/packages/{{ package_file_name }}"
    dest: "{{ temp_path }}/{{ package_file_name }}"
```

## Полный пример исправления

1. Поместите файл пакета в правильную директорию:
   ```bash
   mkdir -p roles/universal_package_installer/files/ubuntu/
   cp bz_sensors_agent-*.deb roles/universal_package_installer/files/ubuntu/
   ```

2. Обновите `vars/ubuntu.yml`:
   ```yaml
   ---
   package_file_name: "bz_sensors_agent-nizhpharm-18f168d8.soc.bi.zone-v2.22.1-BZ_SOC_with_EDR-2025.06.02T09-39-2.deb"
   package_source_path: "ubuntu/{{ package_file_name }}"
   ```

3. Или укажите параметры явно при вызове роли:
   ```yaml
   - hosts: all
     roles:
       - role: universal_package_installer
         vars:
           package_file_name: "bz_sensors_agent-nizhpharm-18f168d8.soc.bi.zone-v2.22.1-BZ_SOC_with_EDR-2025.06.02T09-39-2.deb"
           package_name: "bz_sensors_agent"
   ```

После этих изменений роль должна найти файл пакета и успешно установить его.