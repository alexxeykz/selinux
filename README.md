Что нужно сделать?

Запустить nginx на нестандартном порту 3-мя разными способами: 
1.переключатели setsebool;
2.добавление нестандартного порта в имеющийся тип;
3.формирование и установка модуля SELinux.
К сдаче:
README с описанием каждого решения (скриншоты и демонстрация приветствуются).

2. Обеспечить работоспособность приложения при включенном selinux.
развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
выяснить причину неработоспособности механизма обновления зоны (см. README);
предложить решение (или решения) для данной проблемы;
выбрать одно из решений для реализации, предварительно обосновав выбор;
реализовать выбранное решение и продемонстрировать его работоспособность.

К сдаче:
README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;
исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.


Статус "Принято" ставится при выполнении следующих условий:

для задания 1 описаны, реализованы и продемонстрированы все 3 способа решения;
для задания 2 описана причина неработоспособности механизма обновления зоны;
для задания 2 реализован и продемонстрирован один из способов решения.


```
Запустить nginx на нестандартном порту 3-мя разными способами:
```
README_Selinux
```
Создаём виртуальную машину:
```
```
# -*- mode: ruby -*-
# vim: set ft=ruby :


MACHINES = {
  :selinux => {
        :box_name => "centos/7_2004_01",
  },
}


Vagrant.configure("2") do |config|


  MACHINES.each do |boxname, boxconfig|


      config.vm.define boxname do |box|


        box.vm.box = boxconfig[:box_name]
        box.vm.box_version = boxconfig[:box_version]


        box.vm.host_name = "selinux"
        box.vm.network "forwarded_port", guest: 4881, host: 4881


        box.vm.provider :virtualbox do |vb|
              vb.customize ["modifyvm", :id, "--memory", "1024"]
              needsController = false
        end


        box.vm.provision "shell", inline: <<-SHELL
          #install epel-release
		  yum -y update && yum clean all
          yum install -y epel-release
          #install nginx
          yum install -y nginx
          #change nginx port
          sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
          sed -i 's/listen       80;/listen       4881;/' /etc/nginx/nginx.conf
          #disable SELinux
          #setenforce 0
          #start nginx
          systemctl start nginx
          systemctl status nginx
          #check nginx port
          ss -tlpn | grep 4881
        SHELL
    end
  end
end
```
```
Во время установки видим ошибку:
selinux: Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
```
Получилась созданная машина с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.

заходим на сервер:
```
root@testvm:/home/selinux# vagrant ssh
[vagrant@selinux ~]$ sudo -i
```
1.Запуск nginx на нестандартном порту 3-мя разными способами
```
проверим, что в ОС отключен файервол:
[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
```
Проверим, что конфигурация nginx настроена без ошибок:
```
[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Далее проверим режим работы SELinux:
```
[root@selinux ~]# getenforce
Enforcing
```
Данный режим означает, что SELinux будет блокировать запрещенную активность.

Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool

Находим в логах (/var/log/audit/audit.log) информацию о блокировании порта
```
Нам понадобится утилита sealert, которая проведет аудит файла:
Устанавливаем пакет:
yum install setroubleshoot-server
```
Проверяем
[root@selinux ~]# sealert -a /var/log/audit/audit.log
```
[root@selinux ~]# sealert -a /var/log/audit/audit.log
100% done
found 1 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------

SELinux запрещает /usr/sbin/nginx доступ name_bind к tcp_socket port 4881.

*****  Модуль bind_ports предлагает (точность 92.2)  *************************

Если вы хотите разрешить /usr/sbin/nginx для привязки к сетевому порту $PORT_ЧИСЛО
То you need to modify the port type.
Сделать
# semanage port -a -t PORT_TYPE -p tcp 4881
    где PORT_TYPE может принимать значения: http_cache_port_t, http_port_t, jboss_management_port_t, jboss_messaging_port_t, ntop_port_t, puppet_port_t.

*****  Модуль catchall_boolean предлагает (точность 7.83)  *******************

Если хотите allow nis to enabled
То вы должны сообщить SELinux об этом, включив переключатель «nis_enabled».

Сделать
setsebool -P nis_enabled 1

*****  Модуль catchall предлагает (точность 1.41)  ***************************

Если вы считаете, что nginx должно быть разрешено name_bind доступ к port 4881 tcp_socket по умолчанию.
То рекомендуется создать отчет об ошибке.
Чтобы разрешить доступ, можно создать локальный модуль политики.
Сделать
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp


Дополнительные сведения:
Исходный контекст             system_u:system_r:httpd_t:s0
Целевой контекст              system_u:object_r:unreserved_port_t:s0
Целевые объекты               port 4881 [ tcp_socket ]
Источник                      nginx
Путь к источнику              /usr/sbin/nginx
Порт                          4881
Узел                          <Unknown>
Исходные пакеты RPM           nginx-1.20.1-10.el7.x86_64
Целевые пакеты RPM
Пакет регламента              selinux-policy-3.13.1-268.el7_9.2.noarch
SELinux активен               True
Тип регламента                targeted
Режим                         Enforcing
Имя узла                      selinux
Платформа                     Linux selinux 3.10.0-1127.el7.x86_64 #1 SMP Tue
                              Mar 31 23:36:51 UTC 2020 x86_64 x86_64
Счетчик уведомлений           1
Впервые обнаружено            2024-06-04 22:24:18 UTC
В последний раз               2024-06-04 22:24:18 UTC
Локальный ID                  f8cd023a-2127-4b05-8750-c0ed4888aff5

Построчный вывод сообщений аудита
type=AVC msg=audit(1717539858.692:1089): avc:  denied  { name_bind } for  pid=13719 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0


type=SYSCALL msg=audit(1717539858.692:1089): arch=x86_64 syscall=bind success=no exit=EACCES a0=6 a1=5644f16da8b8 a2=10 a3=7ffc1731d010 items=0 ppid=1 pid=13719 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm=nginx exe=/usr/sbin/nginx subj=system_u:system_r:httpd_t:s0 key=(null)

Hash: nginx,httpd_t,unreserved_port_t,tcp_socket,name_bind
```
```
Итак по проверке модуль выдает нам полезную информацию.
Мы видим, что это Stlinux блокирует порт.
Также видим советы модуля что нужно делать.
```
```
1 - Способ. Прописываем selinux порт 4881:
```
semanage port -a -t http_port_t -p tcp 4881
```
[root@selinux nginx]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux nginx]# systemctl restart nginx

```
```
Проверяем nginx:
[root@selinux nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Вт 2024-06-04 23:05:08 UTC; 1min 54s ago
  Process: 14027 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 14025 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 14024 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 14029 (nginx)
   CGroup: /system.slice/nginx.service
           ├─14029 nginx: master process /usr/sbin/nginx
           └─14031 nginx: worker process

июн 04 23:05:08 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
июн 04 23:05:08 selinux nginx[14025]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
июн 04 23:05:08 selinux nginx[14025]: nginx: configuration file /etc/nginx/nginx.conf test is successful
июн 04 23:05:08 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
```
Возвращаем назад в нерабочее состояние
[root@selinux nginx]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux nginx]# systemctl restart nginx
[root@selinux nginx]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Вт 2024-06-04 23:11:02 UTC; 4s ago
  Process: 14027 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 14049 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 14048 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 14029 (code=exited, status=0/SUCCESS)

июн 04 23:11:02 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
июн 04 23:11:02 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
июн 04 23:11:02 selinux nginx[14049]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
июн 04 23:11:02 selinux nginx[14049]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
июн 04 23:11:02 selinux nginx[14049]: nginx: configuration file /etc/nginx/nginx.conf test failed
июн 04 23:11:02 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
июн 04 23:11:02 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
июн 04 23:11:02 selinux systemd[1]: Unit nginx.service entered failed state.
июн 04 23:11:02 selinux systemd[1]: nginx.service failed.
```
```

Второй способ:
Сделать
setsebool -P nis_enabled 1

Выполняем:
[root@selinux nginx]# setsebool -P nis_enabled 1
[root@selinux nginx]# systemctl restart nginx
[root@selinux nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Вт 2024-06-04 23:16:41 UTC; 3s ago
  Process: 14083 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 14081 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 14080 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 14085 (nginx)
   CGroup: /system.slice/nginx.service
           ├─14085 nginx: master process /usr/sbin/nginx
           └─14087 nginx: worker process

июн 04 23:16:41 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
июн 04 23:16:41 selinux nginx[14081]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
июн 04 23:16:41 selinux nginx[14081]: nginx: configuration file /etc/nginx/nginx.conf test is successful
июн 04 23:16:41 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
```
Возвращаем назад:
[root@selinux nginx]# setsebool -P nis_enabled 0
[root@selinux nginx]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Вт 2024-06-04 23:18:19 UTC; 5s ago
  Process: 14083 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 14104 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 14103 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 14085 (code=exited, status=0/SUCCESS)

июн 04 23:18:19 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
июн 04 23:18:19 selinux nginx[14104]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
июн 04 23:18:19 selinux nginx[14104]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
июн 04 23:18:19 selinux nginx[14104]: nginx: configuration file /etc/nginx/nginx.conf test failed
июн 04 23:18:19 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
июн 04 23:18:19 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
июн 04 23:18:19 selinux systemd[1]: Unit nginx.service entered failed state.
июн 04 23:18:19 selinux systemd[1]: nginx.service failed.
```
```
3-й Способ. 
allow this access for now by executing:
# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
# semodule -i my-nginx.pp

[root@selinux nginx]# ausearch -c 'nginx' --raw | audit2allow -M my-nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i my-nginx.pp

[root@selinux nginx]# semodule -i my-nginx.pp
[root@selinux nginx]# systemctl restart nginx
[root@selinux nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Вт 2024-06-04 23:21:41 UTC; 3s ago
  Process: 14143 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 14140 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 14139 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 14145 (nginx)
   CGroup: /system.slice/nginx.service
           ├─14145 nginx: master process /usr/sbin/nginx
           └─14146 nginx: worker process

июн 04 23:21:41 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
июн 04 23:21:41 selinux nginx[14140]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
июн 04 23:21:41 selinux nginx[14140]: nginx: configuration file /etc/nginx/nginx.conf test is successful
июн 04 23:21:41 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.

Чтобы удалить модуль, надо выполнить команду semodule -r my-nginx. 
Чтобы выключить модуль semodule -v -d my-nginx. Включить модуль semodule -v -e my-nginx
```
```
Проверяем выключаем модуль:
[root@selinux nginx]# semodule -v -d my-nginx
Attempting to disable module 'my-nginx':
Ok: return value of 0.
Committing changes:
Ok: transaction number 0.
[root@selinux nginx]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Вт 2024-06-04 23:47:48 UTC; 13s ago
  Process: 711 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 730 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 729 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 713 (code=exited, status=0/SUCCESS)

июн 04 23:47:48 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
июн 04 23:47:48 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
июн 04 23:47:48 selinux nginx[730]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
июн 04 23:47:48 selinux nginx[730]: nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
июн 04 23:47:48 selinux nginx[730]: nginx: configuration file /etc/nginx/nginx.conf test failed
июн 04 23:47:48 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
июн 04 23:47:48 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
июн 04 23:47:48 selinux systemd[1]: Unit nginx.service entered failed state.
июн 04 23:47:48 selinux systemd[1]: nginx.service failed.
```
```
Включаем назад:
[root@selinux nginx]# semodule -v -e my-nginx
Attempting to enable module 'my-nginx':
Ok: return value of 0.
Committing changes:
Ok: transaction number 0.
[root@selinux nginx]# systemctl restart nginx
[root@selinux nginx]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Вт 2024-06-04 23:49:37 UTC; 2s ago
  Process: 762 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 759 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 758 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 764 (nginx)
   CGroup: /system.slice/nginx.service
           ├─764 nginx: master process /usr/sbin/nginx
           └─766 nginx: worker process

июн 04 23:49:37 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
июн 04 23:49:37 selinux nginx[759]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
июн 04 23:49:37 selinux nginx[759]: nginx: configuration file /etc/nginx/nginx.conf test is successful
июн 04 23:49:37 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```
```
3 варианта успешно протестированы.

```
Задание №2 Обеспечить работоспособность приложения при включенном selinux.
развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
```
Выполним клонирование репозитория: git clone https://github.com/mbfx/otus-linux-adm.git
```
```
root@testvm:/home/selinux_dns# git clone https://github.com/mbfx/otus-linux-adm.git
Клонирование в «otus-linux-adm»...
remote: Enumerating objects: 558, done.
remote: Counting objects: 100% (456/456), done.
remote: Compressing objects: 100% (303/303), done.
remote: Total 558 (delta 125), reused 396 (delta 74), pack-reused 102
Получение объектов: 100% (558/558), 1.38 МиБ | 3.18 МиБ/с, готово.
Определение изменений: 100% (140/140), готово.
```
```
Перейдём в каталог со стендом: cd otus-linux-adm/selinux_dns_problems
Развернём 2 ВМ с помощью vagrant: vagrant up
После того, как стенд развернется, проверим ВМ с помощью команды:
root@testvm:/home/selinux_dns/otus-linux-adm/selinux_dns_problems# vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

```
```
Подключимся к клиенту:
root@testvm:/home/selinux_dns/otus-linux-adm/selinux_dns_problems# vagrant ssh client
```
Попробуем внести изменения в зону: nsupdate -k /etc/named.zonetransfer.key
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit

Cмотрим логи SELinux, чтобы понять в чём дело:

```
[vagrant@client ~]$ sudo -i
[root@client ~]#
[root@client ~]# cat /var/log/audit/audit.log | audit2why
```
Ничего не показало, отсутствуют ошибки.

Подключимся к серверу, не отключаясь от сессии клиента ns01(сервер):
```
[vagrant@ns01 ~]$ sudo -i
```
Посмотрим логи на сервере:

found 2 alerts in /var/log/audit/audit.log
--------------------------------------------------------------------------------
```
Traceback (most recent call last):
  File "/bin/sealert", line 510, in on_analyzer_state_change
    self.output_results()
  File "/bin/sealert", line 524, in output_results
    print siginfo.format_text()
UnicodeEncodeError: 'ascii' codec can't encode characters in position 8-16: ordinal not in range(128)
[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1717687226.045:1833): avc:  denied  { search } for  pid=5642 comm="isc-worker0000" name="net" dev="proc" ino=7077 scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:sysctl_net_t:s0 tclass=dir

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.
				
 Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1717687886.059:1859): avc:  denied  { create } for  pid=5642 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file



```
```
В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t.
Проверим данную проблему в каталоге /etc/named:
```
```
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```

Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. 
Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды:

```
[root@ns01 ~]# sudo semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0
```

Изменим тип контекста безопасности для каталога /etc/named:
```
[root@ns01 ~]# sudo chcon -R -t named_zone_t /etc/named
[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

Попробуем снова внести изменения с клиента:
```
[root@client ~]# nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```
```
[root@client ~]# dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 157
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Jun 06 17:35:31 UTC 2024
;; MSG SIZE  rcvd: 96
```

Видим, что изменения применились.
Перезагружаем и пробуем снова
```
[root@client ~]# reboot
[root@client ~]# dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.15 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 63300
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.                  IN      A

;; ANSWER SECTION:
www.ddns.lab.           60      IN      A       192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.               3600    IN      NS      ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.           3600    IN      A       192.168.50.10

;; Query time: 4 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Thu Jun 06 17:40:07 UTC 2024
;; MSG SIZE  rcvd: 96
```
Все сохранилось. Все работает!!!






