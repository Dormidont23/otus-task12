## Задание № 12. Практика с SELinux ##

1. Запустить nginx на нестандартном порту тремя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.
2. Обеспечить работоспособность приложения при включенном SELinux.
- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
- выяснить причину неработоспособности механизма обновления зоны;
- предложить решение для данной проблемы.

### Запустить nginx на нестандартном порту тремя разными способами ###
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
