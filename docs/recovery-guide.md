# Посібник з відновлення системи

## Дата створення: 15 жовтня 2025
## Система: Proxmox VE + LXC контейнер 222 (debian-docker)

---

## 📋 Структура бекапів

### Proxmox Backup (rootfs контейнера):
- **Шлях:** `/tank/backups/dump/`
- **Формат:** `vzdump-lxc-222-YYYY_MM_DD-HH_MM_SS.tar.zst`
- **Що включає:** rootfs контейнера (local-lvm), конфігурація
- **Що НЕ включає:** mount points (tank/traefik, tank/actual, tank/paperless)

### ZFS Snapshots (datasets):
- **Datasets:** tank/traefik, tank/actual, tank/paperless
- **Формат:** `@autosnap_YYYY-MM-DD_HH:MM:SS_daily/weekly/monthly`
- **Політика:**
    - traefik: 3 daily, 1 weekly
    - actual: 7 daily, 4 weekly, 3 monthly
    - paperless: 7 daily, 4 weekly, 3 monthly

---

## 🔍 Перегляд доступних бекапів

### Список Proxmox бекапів:
```bash
ls -lh /tank/backups/dump/
```

### Список ZFS snapshots:
```bash
# Всі snapshots
zfs list -t snapshot | grep tank

# Тільки для конкретного dataset
zfs list -t snapshot -r tank/traefik
zfs list -t snapshot -r tank/actual
zfs list -t snapshot -r tank/paperless

# З сортуванням за датою
zfs list -t snapshot -S creation | grep tank
```

### Переглянути файли в ZFS snapshot:
```bash
# Загальний синтаксис
ls /tank/DATASET/.zfs/snapshot/SNAPSHOT_NAME/

# Приклад
ls -la /tank/paperless/.zfs/snapshot/autosnap_2025-10-15_01:27:36_daily/
```

### Переглянути що в Proxmox бекапі:
```bash
# Список файлів (перші 100)
tar -I zstd -tvf /tank/backups/dump/vzdump-lxc-222-2025_10_15-00_04_43.tar.zst | head -100

# Повний список (може бути довгим)
tar -I zstd -tvf /tank/backups/dump/vzdump-lxc-222-2025_10_15-00_04_43.tar.zst | less

# Переглянути лог бекапу
cat /tank/backups/dump/vzdump-lxc-222-2025_10_15-00_04_43.log
```

---

## 💾 Відновлення ZFS Datasets

### Спосіб 1: Rollback (повне повернення) ⚠️

**УВАГА:** Видаляє всі зміни після snapshot і всі новіші snapshots!
```bash
# 1. Перевірити які snapshots новіші (БУДУТЬ ВИДАЛЕНІ!)
zfs list -t snapshot -r tank/paperless | grep -A 999 "2025-10-15"

# 2. Створити snapshot поточного стану (на всяк випадок)
zfs snapshot tank/paperless@before-rollback-$(date +%Y%m%d-%H%M)

# 3. Зупинити контейнер
pct stop 222

# 4. Rollback
zfs rollback -r tank/paperless@autosnap_2025-10-15_01:27:36_daily

# 5. Запустити контейнер
pct start 222
```

### Спосіб 2: Відновити окремі файли (безпечно) ✅
```bash
# 1. Зупинити контейнер (рекомендовано)
pct stop 222

# 2. Скопіювати окремий файл
cp /tank/paperless/.zfs/snapshot/autosnap_2025-10-15_01:27:36_daily/important-file.pdf \
   /tank/paperless/important-file-restored.pdf

# 3. Скопіювати директорію
cp -r /tank/paperless/.zfs/snapshot/autosnap_2025-10-15_01:27:36_daily/documents/ \
      /tank/paperless/documents-restored/

# 4. Запустити контейнер
pct start 222
```

### Спосіб 3: Клонувати snapshot (найбезпечніше) 🛡️
```bash
# 1. Створити клон snapshot у новий dataset
zfs clone tank/paperless@autosnap_2025-10-15_01:27:36_daily tank/paperless-restored

# 2. Тепер є два datasets:
#    - tank/paperless (поточний)
#    - tank/paperless-restored (стан від 15 жовтня)

# 3. Можна змонтувати в контейнер як додатковий mount point
pct set 222 -mp3 /tank/paperless-restored,mp=/mnt/paperless-old

# 4. Переглянути, скопіювати що треба

# 5. Видалити клон коли не потрібен
pct set 222 -delete mp3
zfs destroy tank/paperless-restored
```

### Порівняти поточний стан зі snapshot:
```bash
# Показати відмінності
diff -r /tank/paperless/ \
        /tank/paperless/.zfs/snapshot/autosnap_2025-10-15_01:27:36_daily/
```

---

## 🔄 Відновлення LXC контейнера

### Спосіб 1: Restore в новий контейнер (безпечно) ✅
```bash
# 1. Відновити в новий контейнер з ID 223
pct restore 223 /tank/backups/dump/vzdump-lxc-222-2025_10_15-00_04_43.tar.zst \
  --storage local-lvm \
  --hostname debian-docker-restored

# 2. Запустити для перевірки
pct start 223

# 3. Перевірити що все OK
pct enter 223
# (перевірити конфіги, сервіси)
exit

# 4. Якщо OK - перемкнути mount points:
pct stop 222
pct stop 223

# 5. Видалити mount points зі старого контейнера
pct set 222 -delete mp0
pct set 222 -delete mp1
pct set 222 -delete mp2

# 6. Додати mount points до нового контейнера
pct set 223 -mp0 /tank/traefik,mp=/mnt/traefik
pct set 223 -mp1 /tank/actual,mp=/mnt/actual
pct set 223 -mp2 /tank/paperless,mp=/mnt/paperless

# 7. Запустити новий контейнер
pct start 223

# 8. Перевірити що все працює
pct enter 223

# 9. Видалити старий контейнер (коли впевнені)
pct destroy 222
```

### Спосіб 2: Restore поверх існуючого (перезапис) ⚠️

**УВАГА:** Повністю видаляє поточний стан контейнера!
```bash
# 1. Створити snapshot поточного rootfs (на всяк випадок)
zfs snapshot rpool/data/subvol-222-disk-0@before-restore-$(date +%Y%m%d-%H%M)

# 2. Зупинити контейнер
pct stop 222

# 3. Restore з перезаписом
pct restore 222 /tank/backups/dump/vzdump-lxc-222-2025_10_15-00_04_43.tar.zst \
  --force

# 4. Запустити
pct start 222
```

### Спосіб 3: Витягнути окремі файли з бекапу
```bash
# 1. Створити тимчасову директорію
mkdir /tmp/restore-222
cd /tmp/restore-222

# 2. Розпакувати бекап (це займе час!)
tar -I zstd -xvf /tank/backups/dump/vzdump-lxc-222-2025_10_15-00_04_43.tar.zst

# 3. Переглянути структуру
ls -la /tmp/restore-222/

# 4. Скопіювати потрібні файли в контейнер
# Варіант А: через pct push
pct push 222 /tmp/restore-222/root/docker-compose.yml /root/docker-compose-old.yml

# Варіант Б: вручну через pct enter
pct enter 222
# потім cp з /tmp/restore-222/...

# 5. Прибрати тимчасові файли
rm -rf /tmp/restore-222
```

### Витягнути один файл з бекапу (без повного розпакування):
```bash
# Витягнути конкретний файл
tar -I zstd -xvf /tank/backups/dump/vzdump-lxc-222-2025_10_15-00_04_43.tar.zst \
  --strip-components=1 \
  ./root/docker-compose.yml

# Файл з'явиться в поточній директорії
ls -la docker-compose.yml
```

---

## 🔄 Повне відновлення (LXC + Datasets)

**Сценарій:** Повернути ВСЮ систему до стану 15 жовтня 2025
```bash
# 1. Створити snapshots поточного стану (страховка)
zfs snapshot rpool/data/subvol-222-disk-0@before-full-restore-$(date +%Y%m%d-%H%M)
zfs snapshot tank/traefik@before-full-restore-$(date +%Y%m%d-%H%M)
zfs snapshot tank/actual@before-full-restore-$(date +%Y%m%d-%H%M)
zfs snapshot tank/paperless@before-full-restore-$(date +%Y%m%d-%H%M)

# 2. Зупинити контейнер
pct stop 222

# 3. Відновити LXC контейнер в новий ID
pct restore 223 /tank/backups/dump/vzdump-lxc-222-2025_10_15-00_04_43.tar.zst \
  --storage local-lvm

# 4. Відкотити всі datasets до того ж дня/часу
zfs rollback -r tank/traefik@autosnap_2025-10-15_01:27:36_daily
zfs rollback -r tank/actual@autosnap_2025-10-15_01:27:36_daily
zfs rollback -r tank/paperless@autosnap_2025-10-15_01:27:36_daily

# 5. Підключити datasets до нового контейнера
pct set 223 -mp0 /tank/traefik,mp=/mnt/traefik \
             -mp1 /tank/actual,mp=/mnt/actual \
             -mp2 /tank/paperless,mp=/mnt/paperless

# 6. Запустити відновлений контейнер
pct start 223

# 7. Перевірити що все працює
pct enter 223

# 8. Якщо OK - видалити старий контейнер
pct destroy 222

# 9. (Опціонально) Видалити страхові snapshots якщо більше не потрібні
# zfs destroy rpool/data/subvol-222-disk-0@before-full-restore-...
# zfs destroy tank/traefik@before-full-restore-...
# ...
```

---

## 📊 Корисні команди для діагностики

### Перевірити розмір datasets та snapshots:
```bash
# Детальна інформація про простір
zfs list -o space tank/traefik
zfs list -o space tank/actual
zfs list -o space tank/paperless

# Скільки займають всі snapshots
zfs list -t snapshot -o used,name | grep tank
```

### Перевірити статус контейнера:
```bash
# Статус
pct status 222

# Конфігурація
cat /etc/pve/lxc/222.conf

# Логи
pct enter 222
journalctl -xe
```

### Перевірити Sanoid статистику:
```bash
# Моніторинг snapshots
sanoid --monitor-snapshots

# Перевірка здоров'я
sanoid --monitor-health

# Логи sanoid
journalctl -u sanoid.service -n 50
```

### Перевірити останні бекапи:
```bash
# Список бекапів за датою
ls -lht /tank/backups/dump/ | head -10

# Розмір директорії бекапів
du -sh /tank/backups/dump/

# Вільне місце в pool
zfs list tank
zpool list tank
```

---

## ⚠️ Важливі нотатки

### Що ВКЛЮЧЕНО в Proxmox backup:
- ✅ Rootfs контейнера (файлова система `/`)
- ✅ Конфігурація контейнера `/etc/pve/lxc/222.conf`
- ✅ CPU, RAM, мережеві налаштування

### Що НЕ ВКЛЮЧЕНО в Proxmox backup:
- ❌ Дані в `/tank/traefik` (mount point mp0)
- ❌ Дані в `/tank/actual` (mount point mp1)
- ❌ Дані в `/tank/paperless` (mount point mp2)
- ❌ ZFS snapshots datasets

**Для ПОВНОГО відновлення потрібно:**
1. Proxmox backup (rootfs)
2. ZFS snapshots (datasets)

### Розклад автоматичних бекапів:
- **Proxmox vzdump:** щодня о 21:00
- **Sanoid snapshots:** щодня о 00:00 (через systemd timer)

### Політика retention:
- **Proxmox backup:**
    - Keep Last: 3
    - Keep Daily: 7
    - Keep Weekly: 4
    - Keep Monthly: 3

- **ZFS snapshots (traefik):**
    - Daily: 3
    - Weekly: 1

- **ZFS snapshots (actual, paperless):**
    - Daily: 7
    - Weekly: 4
    - Monthly: 3

---

## 🆘 Екстрені сценарії

### Сценарій 1: Видалив важливі файли в контейнері
```bash
# Якщо файли в rootfs:
pct stop 222
pct restore 223 /tank/backups/dump/vzdump-lxc-222-LATEST.tar.zst
pct start 223
# Скопіювати файли з CT 223 в CT 222

# Якщо файли в datasets (traefik/actual/paperless):
ls /tank/paperless/.zfs/snapshot/
cp /tank/paperless/.zfs/snapshot/LATEST/deleted-file.pdf /tank/paperless/
```

### Сценарій 2: Контейнер не запускається
```bash
# Перевірити логи
pct status 222
journalctl -u pve-container@222.service

# Спробувати відновити з останнього бекапу
pct stop 222
pct restore 223 /tank/backups/dump/vzdump-lxc-222-LATEST.tar.zst
pct start 223
```

### Сценарій 3: Помилково змінив конфіги
```bash
# Відкотити dataset з конфігами
pct stop 222
zfs rollback -r tank/traefik@autosnap_LATEST_daily
pct start 222
```

### Сценарій 4: Disk failure / ZFS pool проблеми
```bash
# Перевірити статус pool
zpool status tank

# Якщо pool в порядку але dataset пошкоджено
zpool scrub tank

# Експортувати останній працюючий snapshot поза pool
zfs send tank/paperless@LATEST_daily | gzip > /external-backup/paperless-emergency.zfs.gz
```

---

## 📝 Чеклист перед відновленням

- [ ] Визначити що саме потрібно відновити (rootfs / datasets / все)
- [ ] Знайти потрібний бекап/snapshot за датою
- [ ] Створити snapshot поточного стану (страховка)
- [ ] Зупинити контейнер
- [ ] Виконати відновлення
- [ ] Перевірити що все працює
- [ ] Видалити старі версії/страхові snapshots

---

## 🔗 Корисні посилання

- Proxmox VE Wiki: https://pve.proxmox.com/wiki/
- ZFS Documentation: https://openzfs.github.io/openzfs-docs/
- Sanoid GitHub: https://github.com/jimsalterjrs/sanoid

---

**Створено:** 15 жовтня 2025  
**Система:** Proxmox VE / LXC CT 222 (debian-docker)  
**Останнє оновлення:** 15 жовтня 2025
