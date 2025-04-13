## Занятие 7. Управление пакетами. Дистрибьюция софта

### 1. создать свой RPM (можно взять свое приложение, либо собрать к примеру Apache с определенными опциями)
Для примера возьмем пакет Nginx и соберем его с дополнительным модулем ngx_broli
```
[user@localhost ~]$ sudo yum install -y wget rpmdevtools rpm-build createrepo yum-utils cmake gcc git nano

[root@localhost ~]# mkdir rpm && cd rpm
[root@localhost rpm]# yumdownloader --source nginx
[root@localhost rpm]# rpm -Uvh nginx*.src.rpm
[root@localhost rpm]# yum-builddep nginx
[root@localhost ~]# git clone --recurse-submodules -j8 https://github.com/google/ngx_brotli

[root@localhost brotli]# mkdir out && cd out
[root@localhost out]# cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..
[root@localhost out]# cmake --build . --config Release -j 2 --target brotlienc

[root@localhost rpmbuild]# nano SPECS/nginx.spec
if ! ./configure \
    --prefix=%{_datadir}/nginx \
    --sbin-path=%{_sbindir}/nginx \
    --modules-path=%{nginx_moduledir} \
    --conf-path=%{_sysconfdir}/nginx/nginx.conf \
    --error-log-path=%{_localstatedir}/log/nginx/error.log \
    --http-log-path=%{_localstatedir}/log/nginx/access.log \
    --http-client-body-temp-path=%{_localstatedir}/lib/nginx/tmp/client_body \
    --http-proxy-temp-path=%{_localstatedir}/lib/nginx/tmp/proxy \
    --http-fastcgi-temp-path=%{_localstatedir}/lib/nginx/tmp/fastcgi \
    --http-uwsgi-temp-path=%{_localstatedir}/lib/nginx/tmp/uwsgi \
    --http-scgi-temp-path=%{_localstatedir}/lib/nginx/tmp/scgi \
    --pid-path=/run/nginx.pid \
    --lock-path=/run/lock/subsys/nginx \
    --user=%{nginx_user} \
    --group=%{nginx_user} \
    --with-compat \
    --with-debug \
    --add-module=/root/ngx_brotli \

[root@localhost SPECS]# rpmbuild -ba nginx.spec -D 'debug_package %{nil}'

[root@localhost ~]# ll rpmbuild/RPMS/x86_64/
итого 2252
-rw-r--r--. 1 root root   33744 апр 13 16:26 nginx-1.26.3-1.fc41.x86_64.rpm
-rw-r--r--. 1 root root 1138525 апр 13 16:26 nginx-core-1.26.3-1.fc41.x86_64.rpm
-rw-r--r--. 1 root root  894589 апр 13 16:26 nginx-mod-devel-1.26.3-1.fc41.x86_64.rpm
-rw-r--r--. 1 root root   21544 апр 13 16:26 nginx-mod-http-image-filter-1.26.3-1.fc41.x86_64.rpm
-rw-r--r--. 1 root root   33209 апр 13 16:26 nginx-mod-http-perl-1.26.3-1.fc41.x86_64.rpm
-rw-r--r--. 1 root root   20500 апр 13 16:26 nginx-mod-http-xslt-filter-1.26.3-1.fc41.x86_64.rpm
-rw-r--r--. 1 root root   55675 апр 13 16:26 nginx-mod-mail-1.26.3-1.fc41.x86_64.rpm
-rw-r--r--. 1 root root   89066 апр 13 16:26 nginx-mod-stream-1.26.3-1.fc41.x86_64.rpm

[root@localhost ~]# cp ~/rpmbuild/RPMS/noarch/* ~/rpmbuild/RPMS/x86_64/
[root@localhost ~]# cd rpmbuild/RPMS/x86_64/

[root@localhost x86_64]# yum install ./*.rpm
[root@localhost x86_64]# systemctl start nginx.service
[root@localhost x86_64]# systemctl status nginx.service
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
    Drop-In: /usr/lib/systemd/system/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Sun 2025-04-13 16:33:05 MSK; 5s ago
```

### 2. Создать свой репозиторий и разместить там ранее собранный RPM

```
[root@localhost ~]# mkdir /usr/share/nginx/html/repo

[root@localhost ~]# createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 10 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Pool started (with 5 workers)
Pool finished

[root@localhost ~]# nano /etc/nginx/nginx.conf
    server {
        listen       80;
        listen       [::]:80;
        server_name  _;
        root         /usr/share/nginx/html;
        index index.html index.htm;
        autoindex on;
        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;
    }

[root@localhost ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@localhost ~]# nginx -s reload

[root@localhost ~]# lynx http://localhost/repo
                                                                                                                       Index of /repo/
                                                           Index of /repo/
     ________________________________________________________________________________________________________________________

../
repodata/                                          13-Apr-2025 13:35                   -
nginx-1.26.3-1.fc41.x86_64.rpm                     13-Apr-2025 13:34               33744
nginx-all-modules-1.26.3-1.fc41.noarch.rpm         13-Apr-2025 13:34               10145
nginx-core-1.26.3-1.fc41.x86_64.rpm                13-Apr-2025 13:34             1138525
nginx-filesystem-1.26.3-1.fc41.noarch.rpm          13-Apr-2025 13:34               11893
nginx-mod-devel-1.26.3-1.fc41.x86_64.rpm           13-Apr-2025 13:34              894589
nginx-mod-http-image-filter-1.26.3-1.fc41.x86_6..> 13-Apr-2025 13:34               21544
nginx-mod-http-perl-1.26.3-1.fc41.x86_64.rpm       13-Apr-2025 13:34               33209
nginx-mod-http-xslt-filter-1.26.3-1.fc41.x86_64..> 13-Apr-2025 13:34               20500
nginx-mod-mail-1.26.3-1.fc41.x86_64.rpm            13-Apr-2025 13:34               55675
nginx-mod-stream-1.26.3-1.fc41.x86_64.rpm          13-Apr-2025 13:34               89066
     ________________________________________________________________________________________________________________________

[root@localhost ~]# cat >> /etc/yum.repos.d/otus.repo << EOF
> [otus]
> name=otus-linux
> baseurl=http://localhost/repo
> gpgcheck=0
> enabled=1
> EOF

[root@localhost ~]# dnf repolist --enabled | grep otus
otus                  otus-linux
```

### Добавим пакет в свой репозиторий

```
[root@localhost repo]# wget https://repo.percona.com/yum/percona-release-latest.noarch.rpm
percona-release-late 100% [===================================================================================>]   27.63K    --.-KB/s
                          [Files: 1  Bytes: 27.63K [66.75KB/s] Redirects: 0  Todo: 0  Errors: 0                ]

[root@localhost repo]# createrepo /usr/share/nginx/html/repo/
[root@localhost repo]# dnf config-manager setopt fedora.enabled=0
[root@localhost repo]# dnf config-manager setopt fedora-cisco-openh264.enabled=0
[root@localhost repo]# dnf config-manager setopt updates.enabled=0

[root@localhost repo]# dnf clean all
[root@localhost repo]# dnf makecache
[root@localhost repo]# dnf list --available
Updating and loading repositories:
Repositories loaded.
Available packages
nginx.x86_64                       2:1.26.3-1.fc41 otus
nginx-all-modules.noarch           2:1.26.3-1.fc41 otus
nginx-core.x86_64                  2:1.26.3-1.fc41 otus
nginx-filesystem.noarch            2:1.26.3-1.fc41 otus
nginx-mod-devel.x86_64             2:1.26.3-1.fc41 otus
nginx-mod-http-image-filter.x86_64 2:1.26.3-1.fc41 otus
nginx-mod-http-perl.x86_64         2:1.26.3-1.fc41 otus
nginx-mod-http-xslt-filter.x86_64  2:1.26.3-1.fc41 otus
nginx-mod-mail.x86_64              2:1.26.3-1.fc41 otus
nginx-mod-stream.x86_64            2:1.26.3-1.fc41 otus
percona-release.noarch             1.0-30          otus

[root@localhost repo]# dnf install -y percona-release
Transaction Summary:
 Installing:         1 package

Total size of inbound packages is 28 KiB. Need to download 28 KiB.
After this operation, 49 KiB extra will be used (install 49 KiB, remove 0 B).
[1/1] percona-release-0:1.0-30.noarch                                                         100% |   0.0   B/s |  27.6 KiB |  00m00s
--------------------------------------------------------------------------------------------------------------------------------------
[1/1] Total                                                                                   100% |   0.0   B/s |  27.6 KiB |  00m00s
Running transaction
[1/3] Verify package files                                                                    100% | 500.0   B/s |   1.0   B |  00m00s
[2/3] Prepare transaction                                                                     100% |  66.0   B/s |   1.0   B |  00m00s
[3/3] Installing percona-release-0:1.0-30.noarch                                              100% |  37.0 KiB/s |  50.1 KiB |  00m01s
Warning: skipped PGP checks for 1 package from repository: otus
Complete!

```