
# Занятие 6. NFS

Основная часть: 

- запустить 2 виртуальных машины (сервер NFS и клиента);  
- на сервере NFS должна быть подготовлена и экспортирована директория; 
- в экспортированной директории должна быть поддиректория с именем upload с правами на запись в неё; 
- экспортированная директория должна автоматически монтироваться на клиенте при старте виртуальной машины (systemd, autofs или fstab — любым способом);
- монтирование и работа NFS на клиенте должна быть организована с использованием NFSv3.

```
# on server
# creating shared folder
root@ubuntuserv2402:/home/user# mkdir -p /srv/share/upload
root@ubuntuserv2402:/home/user# chown -R nobody:nogroup /srv/share/
root@ubuntuserv2402:/home/user# chmod 0777 /srv/share/upload/

# specifying ip for client
root@ubuntuserv2402:/home/user# cat << EOF > /etc/exports
> /srv/share 192.168.1.11/24(rw,sync,root_squash)
> EOF
root@ubuntuserv2402:/home/user# cat /etc/exports
/srv/share 192.168.1.11/24(rw,sync,root_squash)
root@ubuntuserv2402:/home/user# exportfs -r
exportfs: /etc/exports [1]: Neither 'subtree_check' or 'no_subtree_check' specified for export "192.168.1.11/24:/srv/share".
  Assuming default behaviour ('no_subtree_check').
  NOTE: this default has changed since nfs-utils version 1.0.x

root@ubuntuserv2402:/home/user# exportfs -s
/srv/share  192.168.1.11/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

# На клиенте
echo "192.168.1.12:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0" >> /etc/fstab

root@ubuntuserv2402:/home/user# systemctl daemon-reload
root@ubuntuserv2402:/home/user# systemctl restart remote-fs.target
root@ubuntuserv2402:/home/user# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=70,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=13138)

# on server
root@ubuntuserv2402:/home/user# cd /srv/share/upload/
root@ubuntuserv2402:/srv/share/upload# touch check_file

# on client
root@ubuntuserv2402:/home/user# ls /mnt/upload
check_file
root@ubuntuserv2402:/home/user# cd /mnt/upload/
root@ubuntuserv2402:/mnt/upload# touch client_file

# on server
root@ubuntuserv2402:/srv/share/upload# ls
check_file  client_file

# on client after reboot
user@ubuntuserv2402:~$ cd /mnt/upload/
user@ubuntuserv2402:/mnt/upload$ ls
check_file  client_file

# on server after reboot
user@ubuntuserv2402:~$ cd /srv/share/upload/
user@ubuntuserv2402:/srv/share/upload$ ls
check_file  client_file
user@ubuntuserv2402:/srv/share/upload$ exportfs -s
exportfs: could not open /var/lib/nfs/.etab.lock for locking: errno 13 (Permission denied)
user@ubuntuserv2402:/srv/share/upload$ sudo exportfs -s
[sudo] password for user:
/srv/share  192.168.1.11/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)
user@ubuntuserv2402:/srv/share/upload$ showmount -a 192.168.1.12
All mount points on 192.168.1.12:
192.168.1.11:/srv/share

# on client after reboot
user@ubuntuserv2402:~$ showmount -a 192.168.1.12
All mount points on 192.168.1.12:
user@ubuntuserv2402:~$ cd /mnt/upload/
user@ubuntuserv2402:/mnt/upload$ mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=68,pgrp=1,timeout=0,minproto=5,max                     proto=5,direct,pipe_ino=4396)
192.168.1.12:/srv/share/ on /mnt type nfs (rw,relatime,vers=3,rsize=524288,wsize                     =524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,mountaddr=192.168.                     1.12,mountvers=3,mountport=43556,mountproto=udp,local_lock=none,addr=192.168.1.1                     2)
user@ubuntuserv2402:/mnt/upload$ ls
check_file  client_file
user@ubuntuserv2402:/mnt/upload$ touch final_check
user@ubuntuserv2402:/mnt/upload$ ls
check_file  client_file  final_check


# on server
# creating server init script
user@ubuntuserv2402:/srv/share/upload$ touch nfss_script.sh
user@ubuntuserv2402:/srv/share/upload$ chmod +x nfss_script.sh
user@ubuntuserv2402:/srv/share/upload$ mv nfss_script.sh /home/user/
user@ubuntuserv2402:/srv/share/upload$ cd
user@ubuntuserv2402:~$ nano nfss_script.sh
user@ubuntuserv2402:~$ cat nfss_script.sh
mkdir -p /srv/share/upload
chown -R nobody:nogroup /srv/share/
chmod 0777 /srv/share/upload/
echo "/srv/share 192.168.1.11/24(rw,sync,root_squash)" >> /etc/exports
exportfs -r

# on client
user@ubuntuserv2402:~$ touch nfsc_script.sh
user@ubuntuserv2402:~$ chmod +x nfsc_script.sh
user@ubuntuserv2402:~$ nano nfsc_script.sh
user@ubuntuserv2402:~$ cat nfsc_script.sh
echo "192.168.1.12:/srv/share/ /mnt nfs vers=3,noauto,x-systemd.automount 0 0" >> /etc/fstab
systemctl daemon-reload
systemctl restart remote-fs.target

```