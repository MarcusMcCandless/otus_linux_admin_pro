## Занятие 9. Инициализация системы. Systemd

### 1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default)
```
root@ubuntuserv2402:~# cat /etc/default/watchlog
# Configuration file for my watchlog service
# Place it to /etc/default

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log


root@ubuntuserv2402:~# echo 'test log first line 12345' > /var/log/watchlog.log
root@ubuntuserv2402:~# echo 'ALERT: very important message' >> /var/log/watchlog.log

root@ubuntuserv2402:~# cat > /opt/watchlog.sh
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi

root@ubuntuserv2402:~# chmod +x /opt/watchlog.sh

root@ubuntuserv2402:~# systemctl start watchlog.timer
root@ubuntuserv2402:~# tail -n 1000 /var/log/syslog  | grep word
2025-04-04T12:06:39.880525+00:00 ubuntuserv2402 root: Пт 04 апр 2025 12:06:39 UTC: I found word, Master!
2025-04-04T12:07:09.901928+00:00 ubuntuserv2402 root: Пт 04 апр 2025 12:07:09 UTC: I found word, Master!

```

### 2. Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта

```
root@ubuntuserv2402:~# apt install spawn-fcgi php php-cgi php-cli  apache2 libapache2-mod-fcgid -y

root@ubuntuserv2402:~# mkdir /etc/spawn-fcgi
root@ubuntuserv2402:~# nano /etc/spawn-fcgi/fcgi.conf
root@ubuntuserv2402:~# systemctl edit --force --full spawn-fcgi.service
Successfully installed edited file '/etc/systemd/system/spawn-fcgi.service'.

root@ubuntuserv2402:~# systemctl start spawn-fcgi.service
root@ubuntuserv2402:~# systemctl status spawn-fcgi.service
● spawn-fcgi.service - Spawn-fcgi startup service by Otus
     Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; preset: enabled)
     Active: active (running) since Sun 2025-04-13 14:37:25 UTC; 6s ago
```

### 3. Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.
```
root@ubuntuserv2402:~# apt install nginx -y

root@ubuntuserv2402:/etc/systemd/system# systemctl edit nginx.service
Successfully installed edited file '/etc/systemd/system/nginx.service.d/override.conf'.
root@ubuntuserv2402:~# cd /etc/nginx/
root@ubuntuserv2402:/etc/nginx# cp nginx.conf nginx-first.conf
root@ubuntuserv2402:/etc/nginx# cp nginx.conf nginx-second.conf
root@ubuntuserv2402:/etc/nginx# nano nginx-first.conf
root@ubuntuserv2402:/etc/nginx# nano nginx-second.conf

root@ubuntuserv2402:/etc/systemd/system# systemctl start nginx@first
root@ubuntuserv2402:/etc/systemd/system# systemctl start nginx@second
root@ubuntuserv2402:/etc/systemd/system# ss -nutpl | grep nginx
tcp   LISTEN 0      511          0.0.0.0:9001       0.0.0.0:*    users:(("nginx",pid=24677,fd=5),("nginx",pid=24676,fd=5),("nginx",pid=24675,fd=5))
tcp   LISTEN 0      511          0.0.0.0:9002       0.0.0.0:*    users:(("nginx",pid=24748,fd=5),("nginx",pid=24747,fd=5),("nginx",pid=24746,fd=5))

root@ubuntuserv2402:/etc/systemd/system# ps afx | grep nginx
  24755 pts/1    S+     0:00                          \_ grep --color=auto nginx
  24675 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-first.conf -g daemon on; master_process on;
  24676 ?        S      0:00  \_ nginx: worker process
  24677 ?        S      0:00  \_ nginx: worker process
  24746 ?        Ss     0:00 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx-second.conf -g daemon on; master_process on;
  24747 ?        S      0:00  \_ nginx: worker process
  24748 ?        S      0:00  \_ nginx: worker process

```