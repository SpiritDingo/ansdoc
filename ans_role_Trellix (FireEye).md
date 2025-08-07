Вот пример Ansible плейбука для удаления Trellix (FireEye), который выполняет указанные действия:

```yaml
---
- name: Удаление Trellix (FireEye)
  hosts: all
  become: yes
  tasks:
    - name: Остановка сервиса xagt
      ansible.builtin.service:
        name: xagt
        state: stopped
      when: "'xagt' in ansible_facts.services"

    - name: Удаление сервиса xagt
      ansible.builtin.package:
        name: xagt
        state: absent
      when: ansible_facts.packages is defined and 'xagt' in ansible_facts.packages

    - name: Удаление каталога /opt/fireeye
      ansible.builtin.file:
        path: /opt/fireeye
        state: absent
```

Пояснения:
1. Первая задача останавливает сервис `xagt`, если он существует в системе
2. Вторая задача удаляет пакет `xagt`, если он установлен
3. Третья задача рекурсивно удаляет каталог `/opt/fireeye` со всем его содержимым

Дополнительные рекомендации:
- Перед выполнением рекомендуется сделать бэкап системы
- Можно добавить обработку ошибок (блоки rescue)
- Для проверки перед удалением можно добавить теги и запускать в --check режиме сначала
- В зависимости от конкретной версии Trellix/FireEye могут потребоваться дополнительные шаги очистки

Плейбук предполагает выполнение с повышенными привилегиями (become: yes).


Вот оптимизированный Ansible-плейбук для удаления Trellix (FireEye) **специально под Oracle Linux и CentOS** (RHEL-подобные системы), который:  
1. Останавливает и удаляет сервис `xagt`,  
2. Удаляет каталог `/opt/fireeye`,  
3. Чистит связанные файлы и зависимости.  

### **Окончательный плейбук** (`remove_trellix.yml`):
```yaml
---
- name: "Удаление Trellix (FireEye) на Oracle Linux/CentOS"
  hosts: all
  become: yes
  tasks:
    - name: "1. Остановка сервиса xagt (если запущен)"
      ansible.builtin.service:
        name: xagt
        state: stopped
        enabled: no
      ignore_errors: yes  # Если сервиса нет, пропускаем ошибку

    - name: "2. Поиск установленного пакета xagt (RPM)"
      ansible.builtin.shell:
        cmd: "rpm -qa | grep -i 'xagt' || true"
      register: xagt_pkg
      changed_when: false

    - name: "3. Удаление пакета xagt (если найден)"
      ansible.builtin.yum:
        name: "{{ xagt_pkg.stdout_lines | first }}"
        state: absent
      when: xagt_pkg.stdout != ""

    - name: "4. Удаление каталога /opt/fireeye (рекурсивно)"
      ansible.builtin.file:
        path: /opt/fireeye
        state: absent

    - name: "5. Дополнительно: удаление конфигов и логов (опционально)"
      ansible.builtin.file:
        path: "{{ item }}"
        state: absent
      loop:
        - /etc/init.d/xagt
        - /var/log/fireeye
        - /var/log/xagt.log
      ignore_errors: yes

    - name: "6. Проверка, что xagt больше не работает"
      ansible.builtin.shell:
        cmd: "pgrep -x xagt || echo 'Service not running'"
      register: xagt_check
      changed_when: false
      failed_when: false

    - name: "7. Вывод результата"
      ansible.builtin.debug:
        msg: "Trellix (FireEye) удалён. Остаточные процессы: {{ xagt_check.stdout }}"
```

### **Что делает этот плейбук?**  
✅ **Для Oracle Linux и CentOS** (использует `yum` вместо `apt`).  
✅ Останавливает и **отключает** сервис `xagt`.  
✅ Ищет и удаляет пакет через `rpm -qa` (надежнее, чем просто `yum remove xagt`).  
✅ **Полностью удаляет `/opt/fireeye`** и связанные файлы (логи, конфиги).  
✅ Проверяет, что процесс `xagt` больше не запущен.  

### **Как запустить?**  
```bash
ansible-playbook -i inventory.ini remove_trellix.yml
```

### **Дополнительно**  
- Если Trellix был установлен через `.deb` (но в Oracle/CentOS это маловероятно), нужно заменить `yum` на `dnf` или доработать под конкретный случай.  
- Можно добавить `tags`, чтобы контролировать этапы (например, пропустить удаление логов).  

Этот вариант **более надёжный**, чем общий, так как учитывает особенности RHEL-систем.
