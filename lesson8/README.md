## Занятие 3-4. Загрузка системы

### 1. Включить отображение меню Grub.

```
root@ubuntuserv2402:~# nano /etc/default/grub

Комментируем строку, скрывающую меню и ставим задержку для выбора пункта меню в 10 секунд.
#GRUB_TIMEOUT_STYLE=hidden
GRUB_TIMEOUT=10

root@ubuntuserv2402:~# update-grub
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-55-generic
Found initrd image: /boot/initrd.img-6.8.0-55-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done

```
![Pasted image 20250405211440.png](https://github.com/MarcusMcCandless/otus_linux_admin_pro/blob/main/lesson8/Pasted%20image%2020250405211440.png)

### Способ 1. init=/bin/bash
![VirtualBox_ubuntu_serv2402_02_04_2025_17_02_32.png](https://github.com/MarcusMcCandless/otus_linux_admin_pro/blob/main/lesson8/VirtualBox_ubuntu_serv2402_02_04_2025_17_02_32.png)

![VirtualBox_ubuntu_serv2402_02_04_2025_17_05_12.png](https://github.com/MarcusMcCandless/otus_linux_admin_pro/blob/main/lesson8/VirtualBox_ubuntu_serv2402_02_04_2025_17_05_12.png)
### Способ 2. Recovery mode
![VirtualBox_ubuntu_serv2402_02_04_2025_17_08_54.png](https://github.com/MarcusMcCandless/otus_linux_admin_pro/blob/main/lesson8/VirtualBox_ubuntu_serv2402_02_04_2025_17_08_54.png)

![VirtualBox_ubuntu_serv2402_02_04_2025_17_09_13.png](https://github.com/MarcusMcCandless/otus_linux_admin_pro/blob/main/lesson8/VirtualBox_ubuntu_serv2402_02_04_2025_17_09_13.png)
## Установить систему с LVM, после чего переименовать VG

```
root@ubuntuserv2402:~# vgs
  VG        #PV #LV #SN Attr   VSize   VFree  
  ubuntu-vg   1   2   0 wz--n- <23,00g <13,00g
  vg_var      2   1   0 wz--n-   3,99g   2,12g

root@ubuntuserv2402:~# vgrename ubuntu-vg ubuntu-otus
root@ubuntuserv2402:~# vgs
  VG          #PV #LV #SN Attr   VSize   VFree  
  ubuntu-otus   1   2   0 wz--n- <23,00g <13,00g
  vg_var        2   1   0 wz--n-   3,99g   2,12g



root@ubuntuserv2402:~# update-grub
/usr/sbin/grub-probe: error: failed to get canonical path of `/dev/mapper/ubuntu—vg-ubuntu--lv'.

root@ubuntuserv2402:/etc/grub.d# reboot

при загрузке в grub нажимаем E
edit linux ubuntu--vg ---> ubuntu--otus ; Ctrl+X

root@ubuntuserv2402:~# update-grub
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.8.0-55-generic
Found initrd image: /boot/initrd.img-6.8.0-55-generic
Warning: os-prober will not be executed to detect other bootable partitions.
Systems on them will not be added to the GRUB boot configuration.
Check GRUB_DISABLE_OS_PROBER documentation entry.
Adding boot menu entry for UEFI Firmware Settings ...
done

```
