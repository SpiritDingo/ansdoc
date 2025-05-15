Если вам нужно смонтировать NFS-папку с Windows-сервера в **openSUSE**, а доступ должен быть под **доменным пользователем** (Active Directory), то процесс немного сложнее, так как NFS сам по себе не интегрируется с AD. Вот пошаговая инструкция:

---

## **1. Настройка NFS на Windows**
### **1.1. Установка роли NFS-сервера**
- Откройте **"Управление сервером"** → **"Добавить роли и компоненты"**.
- Выберите **"Служба NFS"** → **"Сервер NFS"**.
- Убедитесь, что служба **Server for NFS** запущена.

### **1.2. Настройка общей папки**
- Кликните правой кнопкой на папке → **"Свойства"** → вкладка **"NFS-доступ"**.
- Нажмите **"Управление доступом NFS"**.
- Добавьте права:
  - **"Тип доступа"**: Read-Write или Read-Only.
  - **"Код пользователя"** (`UID`) и **"Код группы"** (`GID`) должны соответствовать пользователю в Linux (см. ниже).

### **1.3. Настройка анонимного доступа (если нет Kerberos)**
Поскольку NFS в Windows плохо работает с AD напрямую, можно:
- Разрешить анонимный доступ с маппингом на определенный UID/GID.
- В **PowerShell** (от админа):
  ```powershell
  Set-NfsMappingStore -EnableLdapMapping $false
  nfsadmin server config sec=sys
  ```
  Это заставит NFS использовать локальные UID/GID вместо AD.

---

## **2. Настройка Linux (openSUSE)**
### **2.1. Установка NFS-клиента**
```bash
sudo zypper install nfs-client
```

### **2.2. Получение UID/GID доменного пользователя**
Перед монтированием нужно узнать, какой `UID` и `GID` имеет ваш доменный пользователь в Linux.  
Проверьте:
```bash
id username@domain
```
Пример вывода:
```bash
uid=10000(username) gid=10000(domain users) groups=...
```
Запомните `UID` и `GID`.

### **2.3. Монтирование с указанием UID/GID**
```bash
sudo mkdir -p /mnt/nfs_win
sudo mount -t nfs <Windows_IP>:/shared /mnt/nfs_win -o vers=3,nolock,uid=10000,gid=10000
```
- `uid=10000,gid=10000` — замените на свои значения из `id`.

### **2.4. Автомонтирование через `/etc/fstab`**
Добавьте строку:
```bash
<Windows_IP>:/shared  /mnt/nfs_win  nfs  vers=3,nolock,uid=10000,gid=10000,noauto,x-systemd.automount  0  0
```
Примените:
```bash
sudo mount -a
```

---

## **3. Альтернатива: Kerberos + NFSv4 (если нужна безопасность)**
Если требуется безопасный доступ через AD, можно настроить **NFSv4 + Kerberos**:
1. **На Windows**:
   - Включите **"Kerberos authentication"** в настройках NFS.
   ```powershell
   nfsadmin server config sec=krb5
   ```
2. **На openSUSE**:
   - Настройте Kerberos-аутентификацию (`kinit user@DOMAIN`).
   - Монтируйте с `sec=krb5`:
     ```bash
     sudo mount -t nfs4 <Windows_IP>:/shared /mnt/nfs_win -o sec=krb5,vers=4.2
     ```

---

## **4. Возможные проблемы и решения**
### **Ошибка: "Permission denied"**
- Убедитесь, что на Windows в NFS-шаре указаны правильные `UID`/`GID`.
- Попробуйте:
  ```bash
  sudo mount -t nfs <IP>:/share /mnt/nfs_win -o vers=3,nolock,uid=10000,gid=10000,rw
  ```

### **Ошибка: "RPC timeout"**
- Проверьте, открыты ли порты **2049 (TCP)** и **111 (UDP)** в брандмауэре Windows.

### **Ошибка: "Stale file handle"**
- Перемонтируйте:
  ```bash
  sudo umount -l /mnt/nfs_win
  sudo mount -a
  ```

---

## **Итог**
- **Простой способ** (без Kerberos):  
  ```bash
  sudo mount -t nfs <Windows_IP>:/shared /mnt/nfs_win -o vers=3,nolock,uid=10000,gid=10000
  ```
- **Безопасный способ** (с Kerberos):  
  ```bash
  sudo mount -t nfs4 <Windows_IP>:/shared /mnt/nfs_win -o sec=krb5,vers=4.2
  ```

Если нужно, чтобы права сохранялись правильно, убедитесь, что UID/GID в Linux совпадают с маппингом на Windows. 🚀