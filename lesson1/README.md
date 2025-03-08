
## Занятие 1. Обновление ядра системы

### 1. Запустить ВМ.

Установил Fedora Server 41

### 2. Обновить ядро ОС на новейшую стабильную версию из mainline-репозитория.

```
# проверим текущую версию ядра
[user@localhost ~]$ uname -r
6.11.4-301.fc41.x86_64

# добавляем mainline репозиторий
sudo dnf -y copr enable @kernel-vanilla/mainline
# устанавливаем последнюю версию ядра
sudo dnf upgrade 'kernel*'

# Проверяем, что ядро появилось в /boot.
[user@localhost ~]$ ls -al /boot/*vanilla*
-rw-r--r--. 1 root root   282986 мар  6 03:00 /boot/config-6.14.0-0.rc5.20250306gt848e0763.345.vanilla.fc41.x86_64
-rw-------. 1 root root 28262788 мар  6 23:17 /boot/initramfs-6.14.0-0.rc5.20250306gt848e0763.345.vanilla.fc41.x86_64.img
-rw-r--r--. 1 root root   186036 мар  6 23:17 /boot/symvers-6.14.0-0.rc5.20250306gt848e0763.345.vanilla.fc41.x86_64.xz
-rw-r--r--. 1 root root 11099721 мар  6 03:00 /boot/System.map-6.14.0-0.rc5.20250306gt848e0763.345.vanilla.fc41.x86_64
-rwxr-xr-x. 1 root root 16730360 мар  6 03:00 /boot/vmlinuz-6.14.0-0.rc5.20250306gt848e0763.345.vanilla.fc41.x86_64

# После перезагрузки
[user@localhost ~]$ uname -r
6.14.0-0.rc5.20250306gt848e0763.345.vanilla.fc41.x86_64
```


