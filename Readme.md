Vagrant стенд для NFS

Цель:

развернуть сервис NFS и подключить к нему клиента;

Описание/Пошаговая инструкция выполнения домашнего задания:

Для выполнения домашнего задания используйте методичку

Vagrant стенд для NFS

    vagrant up должен поднимать 2 виртуалки: сервер и клиент;
    на сервер должна быть расшарена директория;
    на клиента она должна автоматически монтироваться при старте (fstab или autofs);
    в шаре должна быть папка upload с правами на запись;
    требования для NFS: NFSv3 по UDP, включенный firewall.
    
 _____
 1) Подключаемся к серверу:
 stas@myserv:~/nfs$ vagrant ssh nfss
 
 2) Устанавливаем утилиты:
 [vagrant@nfss ~]$ sudo yum install nfs-utils
 Complete!
 
 3) включаем firewall и проверяем, что он работает:
 [root@nfss ~]# systemctl enable firewalld --now 
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.

4) разрешаем в firewall доступ к сервисам NFS
[root@nfss ~]# firewall-cmd --add-service="nfs3" \
> --add-service="rpc-bind" \
> --add-service="mountd" \
> --permanent 
success

[root@nfss ~]# firewall-cmd --reload
success

5) включаем сервер NFS 
[root@nfss ~]# systemctl enable nfs --now
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.

6) проверяем наличие слушаемых портов 
[root@nfss ~]# ss -tnplu
tcp   LISTEN     0      128                  *:111                              *:*                   users:(("rpcbind",pid=377,fd=8))

7) создаём и настраиваем директорию, которая будет экспортирована в будущем
[root@nfss ~]# mkdir -p /srv/share/upload
[root@nfss ~]# chown -R nfsnobody:nfsnobody /srv/share
[root@nfss ~]# chmod 0777 /srv/share/upload

8) создаём в файле /etc/exports структуру, которая позволит экспортировать ранее созданную директорию 
[root@nfss~]# cat << EOF > /etc/exports 
> /srv/share 192.168.50.11/32(rw,sync,root_squash)
> EOF

9) экспортируем ранее созданную директорию 
[root@nfss ~]# exportfs -r

10) проверяем экспортированную директорию
[root@nfss ~]# exportfs -s
/srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

11) Заходим на клиент:
stas@myserv:~/nfs$ vagrant ssh nfsc

12) доустановим вспомогательные утилиты
[root@nfsc ~]# yum install nfs-utils
[root@nfsc ~]# yum install nfs-utils


13) включаем firewall 
[root@nfsc ~]# systemctl enable firewalld --now 
Created symlink from /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service to /usr/lib/systemd/system/firewalld.service.
Created symlink from /etc/systemd/system/multi-user.target.wants/firewalld.service to /usr/lib/systemd/system/firewalld.service.

[root@nfsc ~]# systemctl status firewalld
Active: active (running) 

14) добавляем в /etc/fstab строку
[root@nfsc ~]# echo "192.168.50.10:/srv/share/ /mnt nfs vers=3,proto=udp,noauto,x-systemd.automount 0 0" >> /etc/fstab

[root@nfsc ~]# systemctl daemon-reload 
[root@nfsc ~]# systemctl restart remote-fs.target

15) заходим в директорию /mnt/
[root@nfsc ~]# mount | grep mnt
systemd-1 on /mnt type autofs (rw,relatime,fd=46,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=59682)

16) Версия 3 отключена на уровне ядра

17) Убираем версию 3 из конфига
[root@nfsc ~]# nano /etc/fstab
[root@nfsc ~]# systemctl daemon-reload
[root@nfsc ~]# systemctl restart remote-fs.target

[root@nfsc ~]# mount | grep /tst
systemd-1 on /tst type autofs (rw,relatime,fd=46,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=60924)

18) заходим на сервер 
stas@myserv:~/nfs$ vagrant ssh nfss

19) заходим в каталог
[vagrant@nfss ~]$ cd /srv/share/upload

20) создаём тестовый файл
[vagrant@nfss upload]$ touch check_file
[vagrant@nfss upload]$ dir
check_file

21) заходим на клиент
stas@myserv:~/nfs$ vagrant ssh nfsc

22) заходим в каталог 
[root@nfsc mntt]# cd /mntt/upload

23) проверяем наличие ранее созданного файла
[root@nfsc upload]# dir
check_file

24) создаём тестовый файл 
[root@nfsc upload]# touch client_file

25) проверяем, что файл успешно создан 
[root@nfsc upload]# dir
check_file  client_file

26) перезагружаем клиент 
[root@nfsc ~]# reboot now
Connection to 127.0.0.1 closed by remote host.

27) заходим на клиент
stas@myserv:~/nfs$ vagrant ssh nfsc

28) проверяем работу RPC
[vagrant@nfsc ~]$ showmount -a 192.168.50.10
All mount points on 192.168.50.10:

29) аходим в каталог
[vagrant@nfsc mntt]$ cd /mntt/upload

30) проверяем статус монтирования
[vagrant@nfsc upload]$ mount | grep mntt
systemd-1 on /mntt type autofs (rw,relatime,fd=32,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=11162)
192.168.50.10:/srv/share/ on /mntt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.50.10)

31) проверяем наличие ранее созданных файлов 
[vagrant@nfsc upload]$ dir
check_file  client_file

32) создаём тестовый файл
[vagrant@nfsc upload]$ touch final_check
[vagrant@nfsc upload]$ dir
check_file  client_file  final_check

33) заходим на сервер
stas@myserv:~/nfs$ vagrant ssh nfss

34) перезагружаем сервер
[vagrant@nfss ~]$ sudo -i
[root@nfss ~]# reboot now
Connection to 127.0.0.1 closed by remote host.

35) заходим на сервер 
stas@myserv:~/nfs$ vagrant ssh nfs

36) проверяем наличие файлов в каталоге
[vagrant@nfss ~]$ dir /srv/share/upload/
check_file  client_file  final_check

37) проверяем статус сервера NFS
[vagrant@nfss ~]$ systemctl status nfs
Active: active (exited)

38) проверяем статус firewall
[vagrant@nfss ~]$ systemctl status firewalld
Active: active (running)

39) проверяем экспорты
[root@nfss ~]# exportfs -s
/srv/share  192.168.50.11/32(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,root_squash,no_all_squash)

40) проверяем работу RPC 
[root@nfss ~]# showmount -a 192.168.50.10
All mount points on 192.168.50.10:
192.168.50.11:/srv/share

41) перезагружаем клиент 
[root@nfsc ~]# reboot now
Connection to 127.0.0.1 closed by remote host.

42) заходим на клиент
stas@myserv:~/nfs$ vagrant ssh nfsc


43) проверяем работу RPC 
[vagrant@nfsc ~]$ showmount -a 192.168.50.10
All mount points on 192.168.50.10:

44) заходим в каталог
[vagrant@nfsc ~]$ cd /mntt/upload

45) проверяем статус монтирования
[vagrant@nfsc upload]$ mount | grep mntt
systemd-1 on /mntt type autofs (rw,relatime,fd=27,pgrp=1,timeout=0,minproto=5,maxproto=5,direct,pipe_ino=10915)
192.168.50.10:/srv/share/ on /mntt type nfs (rw,relatime,vers=3,rsize=32768,wsize=32768,namlen=255,hard,proto=udp,timeo=11,retrans=3,sec=sys,mountaddr=192.168.50.10,mountvers=3,mountport=20048,mountproto=udp,local_lock=none,addr=192.168.50.10)

46) проверяем наличие ранее созданных файлов 
[vagrant@nfsc upload]$ dir
check_file  client_file  final_check

47) создаём тестовый файл 
[vagrant@nfsc upload]$ touch final_check_2

48) проверяем, что файл успешно создан 
[vagrant@nfsc upload]$ dir
check_file  client_file  final_check  final_check_2









 
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
