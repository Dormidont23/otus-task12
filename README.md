## Задание № 12. Практика с SELinux ##

Запустить nginx на нестандартном порту тремя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.

Во время развёртывания стенда, при попытке запуска nginx, выдаётся ошибка. Причина - нестандартный порт nginx.
```
otus-task12: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
otus-task12: ● nginx.service - The nginx HTTP and reverse proxy server
otus-task12:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
otus-task12:    Active: failed (Result: exit-code) since Mon 2024-01-15 13:42:24 UTC; 18ms ago
otus-task12:   Process: 2557 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
otus-task12:   Process: 2556 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
otus-task12:
otus-task12: Jan 15 13:42:24 otus-task12 systemd[1]: Starting The nginx HTTP and reverse proxy server...
otus-task12: Jan 15 13:42:24 otus-task12 nginx[2557]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
otus-task12: Jan 15 13:42:24 otus-task12 nginx[2557]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
otus-task12: Jan 15 13:42:24 otus-task12 nginx[2557]: nginx: configuration file /etc/nginx/nginx.conf test failed
otus-task12: Jan 15 13:42:24 otus-task12 systemd[1]: nginx.service: control process exited, code=exited status=1
otus-task12: Jan 15 13:42:24 otus-task12 systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
otus-task12: Jan 15 13:42:24 otus-task12 systemd[1]: Unit nginx.service entered failed state.
otus-task12: Jan 15 13:42:24 otus-task12 systemd[1]: nginx.service failed.
```
Убедимся, что отключен файервол и проверим правильность конфигурации nginx:\
[root@otus-task12 ~]# **systemctl status firewalld**\
● firewalld.service - firewalld - dynamic firewall daemon\
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)\
   Active: inactive (dead)\
     Docs: man:firewalld(1)\
[root@otus-task12 ~]# **nginx -t**\
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok\
nginx: configuration file /etc/nginx/nginx.conf test is successful

Проверим режим работы SELinux:\
[root@otus-task12 ~]# **getenforce**\
Enforcing
#### Разрешить в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool ####
С помощью утилиты **audit2why** посмотрим, почему трафик блокируется.\
[root@otus-task12 ~]# **cat /var/log/audit/audit.log | grep 4881 | audit2why**
```
type=AVC msg=audit(1705326144.424:693): avc:  denied  { name_bind } for  pid=2557 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1
```
Утилита подсказывает, что нужно поменять параметр nis_enabled:\
[root@otus-task12 ~]# **setsebool -P nis_enabled 1**

Перезапустим nginx и посмотрим на его статус:\
[root@otus-task12 ~]# **systemctl restart nginx**\
[root@otus-task12 ~]# **systemctl status nginx**
```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2024-01-15 14:49:36 UTC; 7s ago
  Process: 1482 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 1480 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 1478 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 1484 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1484 nginx: master process /usr/sbin/nginx
           └─1486 nginx: worker process

Jan 15 14:49:36 otus-task12 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 15 14:49:36 otus-task12 nginx[1480]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 15 14:49:36 otus-task12 nginx[1480]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 15 14:49:36 otus-task12 systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Также работу nginx можно проверить, например, телнетом:\
[root@otus-task12 ~]# **telnet 127.0.0.1 4881**\
Trying 127.0.0.1...\
Connected to 127.0.0.1.\
Escape character is '^]'.\
^]\
telnet> q\
Connection closed.

Статус параметра nis_enabled можно проверить командой:\
[root@otus-task12 ~]# **getsebool -a | grep nis_enabled**\
nis_enabled --> on

Вернём всё, как было, т.е. отключим параметр:\
[root@otus-task12 ~]# **setsebool -P nis_enabled off**

При попытке перезапапустить nginx, ничего не получается:\
[root@otus-task12 ~]# **systemctl restart nginx**\
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
#### Разрешить в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип ####
Поиск имеющегося типа, для http трафика:\
[root@otus-task12 ~]# **semanage port -l | grep http**
```
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```
Добавим порт 4881 в тип http_port_t и посмотрим, что получилось:\
[root@otus-task12 ~]# **semanage port -a -t http_port_t -p tcp 4881**\
[root@otus-task12 ~]# **semanage port -l | grep  http_port_t**
```
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```
Теперь перезапустим nginx и проверим его работу:\
[root@otus-task12 ~]# **systemctl restart nginx**\
[root@otus-task12 ~]# **systemctl status nginx**
```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-01-16 06:57:16 UTC; 6s ago
  Process: 1481 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 1479 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 1477 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 1483 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1483 nginx: master process /usr/sbin/nginx
           └─1485 nginx: worker process

Jan 16 06:57:16 otus-task12 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 16 06:57:16 otus-task12 nginx[1479]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 16 06:57:16 otus-task12 nginx[1479]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 16 06:57:16 otus-task12 systemd[1]: Started The nginx HTTP and reverse proxy server.
```
Проверим работу nginx телнетом:\
[root@otus-task12 ~]# **telnet 127.0.0.1 4881**\
Trying 127.0.0.1...\
Connected to 127.0.0.1.\
Escape character is '^]'.\
^]\
telnet> q\
Connection closed.

~Сломаем~ Вернём всё, как было и убедимся, что nginx не работает:\
[root@otus-task12 ~]# **semanage port -d -t http_port_t -p tcp 4881**\
[root@otus-task12 ~]# **semanage port -l | grep  http_port_t**
```
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
```
[root@otus-task12 ~]# **systemctl restart nginx**\
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.\
[root@otus-task12 ~]# **systemctl status nginx**
```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Tue 2024-01-16 07:04:25 UTC; 13s ago
  Process: 1481 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 1521 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 1520 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 1483 (code=exited, status=0/SUCCESS)

Jan 16 07:04:25 otus-task12 systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jan 16 07:04:25 otus-task12 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 16 07:04:25 otus-task12 nginx[1521]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 16 07:04:25 otus-task12 nginx[1521]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
Jan 16 07:04:25 otus-task12 nginx[1521]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 16 07:04:25 otus-task12 systemd[1]: nginx.service: control process exited, code=exited status=1
Jan 16 07:04:25 otus-task12 systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
Jan 16 07:04:25 otus-task12 systemd[1]: Unit nginx.service entered failed state.
Jan 16 07:04:25 otus-task12 systemd[1]: nginx.service failed.
```
#### Разрешить в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux ####
Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту:\
[root@otus-task12 ~]# **grep nginx /var/log/audit/audit.log | audit2allow -M nginx**\
******************** IMPORTANT ***********************\
To make this policy package active, execute:

semodule -i nginx.pp

Модуль сформировался. Применим его:\
[root@otus-task12 ~]# **semodule -i nginx.pp**

Попробуем запустить nginx:\
[root@otus-task12 ~]# **systemctl start nginx**\
[root@otus-task12 ~]# **systemctl status nginx**
```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-01-16 07:18:13 UTC; 6s ago
  Process: 1575 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 1573 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 1572 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 1577 (nginx)
   CGroup: /system.slice/nginx.service
           ├─1577 nginx: master process /usr/sbin/nginx
           └─1579 nginx: worker process

Jan 16 07:18:13 otus-task12 systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 16 07:18:13 otus-task12 nginx[1573]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 16 07:18:13 otus-task12 nginx[1573]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 16 07:18:13 otus-task12 systemd[1]: Started The nginx HTTP and reverse proxy server.
```
