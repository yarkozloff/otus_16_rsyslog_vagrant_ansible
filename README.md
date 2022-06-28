# Централизованный сбор логов

### Описание/Пошаговая инструкция выполнения домашнего задания:
1) В вагранте поднимаем 2 машины web и log
2) на web поднимаем nginx
3) на log настраиваем центральный лог сервер на любой системе на выбор
- journald;
- rsyslog;
- elk.
4) настраиваем аудит, следящий за изменением конфигов нжинкса Все критичные логи с web должны собираться и локально и удаленно. Все логи с nginx должны уходить на удаленный сервер (локально только критичные). Логи аудита должны также уходить на удаленную систему. Формат сдачи ДЗ - vagrant + ansible
- развернуть еще машину elk
- таким образом настроить 2 центральных лог системы elk и какую либо еще;
- в elk должны уходить только логи нжинкса;
- во вторую систему все остальное.

### В вагранте поднимаем 2 машины web и log
Используем локально загруженный box с centos7, настраиваем Vagrantfile:
```
Vagrant.configure("2") do |config|
    # Base VM OS configuration.
    config.vm.box = "centos7"
    config.vm.provider :virtualbox do |v|
        v.memory = 512
        v.cpus = 1
    end
    # Define two VMs with static private IP addresses.
    boxes = [
        { :name => "web",
          :ip => "192.168.50.10",
        },
        { :name => "log",
        :ip => "192.168.50.11",
        }
    ]
    # Provision each of the VMs.
    boxes.each do |opts|
     config.vm.define opts[:name] do |config|
        config.vm.hostname = opts[:name]
        config.vm.network "private_network", ip: opts[:ip]
        end
    end
end
```
Подключаемся к машинам, указываем Московское время. Пересоздаем файл /etc/localtime с значением Europe/Moscow
```
[root@log ~]# mv /etc/localtime /etc/localtime.bak
[root@log ~]# ln -s /usr/share/zoneinfo/Europe/Moscow /etc/localtime
[root@log ~]# date
Thu Jun 23 23:41:08 MSK 2022
```
### На web поднимаем nginx
На сервере web устанавливаем epel-release, nginx. Проверяем статус службы (пришлось запустить):
```
[root@web ~]# systemctl start nginx
[root@web ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-06-23 23:48:05 MSK; 5s ago
  Process: 24716 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 24714 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  ...
[root@web ~]# ss -tln | grep 80
LISTEN     0      128          *:80                       *:*
LISTEN     0      128       [::]:80                    [::]:*
```
Также проверяем на хосте:
![image](https://user-images.githubusercontent.com/69105791/175397367-219a89f7-bf0a-47c0-8111-2e7e2cf47d34.png)

### на log настраиваем центральный лог сервер

Подключаемся к машине log и проверяем существование/версию rsyslog:
```
[root@log ~]#  yum list rsyslog
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: centos-mirror.rbc.ru
 * extras: centos-mirror.rbc.ru
 * updates: centos-mirror.rbc.ru
Installed Packages
rsyslog.x86_64                              8.24.0-52.el7                                  @anaconda
Available Packages
rsyslog.x86_64                              8.24.0-57.el7_9.3                              updates
```
Правим /etc/rsyslog.conf. Добавив строки (открываем порт 514 и правила приёма сообщений от хостов):
```
# provides UDP syslog reception
module(load="imudp")
input(type="imudp" port="514")

# provides TCP syslog reception
module(load="imtcp")
input(type="imtcp" port="514")


$template RemoteLogs,"/var/log/rsyslog/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& ~
```
Данные параметры будут отправлять в папку /var/log/rsyslog логи, которые будут приходить от других серверов. Например, Access-логи nginx от сервера web, будут идти в файл /var/log/rsyslog/web/nginx_access.log

Далее сохраняем файл и перезапускаем службу rsyslog: systemctl restart rsyslog

Наблюдаем открытые порты:
```
[root@log ~]# ss -tuln | grep 514
udp    UNCONN     0      0         *:514                   *:*
udp    UNCONN     0      0      [::]:514                [::]:*
tcp    LISTEN     0      25        *:514                   *:*
tcp    LISTEN     0      25     [::]:514                [::]:*
```
### Настройка отправки логов
Подключаемся на машину web, проверяем версию nginx (должна быть старше 1.7):
```
[root@web ~]# rpm -qa | grep nginx
nginx-1.20.1-9.el7.x86_64
nginx-filesystem-1.20.1-9.el7.noarch
```
Находим в файле /etc/nginx/nginx.conf раздел с логами и приводим их к следующему виду:
```
    error_log /var/log/nginx/error.log;
    error_log syslog:server=192.168.50.11:514,tag=nginx_error;
    access_log syslog:server=192.168.50.11:514,tag=nginx_access,severity=info combined;
```
** Tag нужен для того, чтобы логи записывались в разные файлы.
По умолчанию, error-логи отправляют логи, которые имеют severity: error, crit, alert и emerg.
Если трубуется хранили или пересылать логи с другим severity, то это также можно указать в настройках nginx.
Далее проверяем, что конфигурация nginx указана правильно: nginx -t
```
[root@web ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Далее перезапустим nginx: systemctl restart nginx

Чтобы проверить, что логи ошибок также улетают на удаленный сервер, можно удалить картинку, к которой будет обращаться nginx во время открытия веб-сраницы:
rm /usr/share/nginx/html/img/header-background.png

Попробуем несколько раз зайти по адресу http://192.168.50.10

Далее заходим на log-сервер и смотрим информацию об nginx:
```[root@log ~]# cat /var/log/rsyslog/web/nginx_error.log
Jun 24 00:24:19 web nginx_error: 2022/06/24 00:24:19 [error] 24772#24772: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.50.1, server: _, request: "GET /favicon.ico HTTP/1.1", host: "192.168.50.10", referrer: "http://192.168.50.10/"


[root@log ~]# cat /var/log/rsyslog/web/nginx_access.log
Jun 24 00:24:19 web nginx_access: 192.168.50.1 - - [24/Jun/2022:00:24:19 +0300] "GET / HTTP/1.1" 200 4833 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:101.0) Gecko/20100101 Firefox/101.0"
Jun 24 00:24:19 web nginx_access: 192.168.50.1 - - [24/Jun/2022:00:24:19 +0300] "GET /favicon.ico HTTP/1.1" 404 3650 "http://192.168.50.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:101.0) Gecko/20100101 Firefox/101.0"
```

### Настройка аудита, контролирующего изменения конфигурации nginx
За аудит отвечает утилита auditd (на сервере web), в RHEL-based системах обычно он уже предустановлен. Проверим:
```
[root@web ~]# rpm -qa | grep audit
audit-2.8.5-4.el7.x86_64
audit-libs-2.8.5-4.el7.x86_64
```
Добавим правило, которое будет отслеживать изменения в конфигруации nginx. Для этого в конец файла /etc/audit/rules.d/audit.rules добавим следующие строки:
```
-w /etc/nginx/nginx.conf -p wa -k nginx_conf
-w /etc/nginx/default.d/ -p wa -k nginx_conf
```
Данные правила позволяют контролировать запись (w) и измения атрибутов (a) в:
● /etc/nginx/nginx.conf
● Всех файлов каталога /etc/nginx/default.d/
Для более удобного поиска к событиям добавляется метка nginx_conf

Перезапускаем службу auditd: service auditd restart

После данных изменений у нас начнут локально записываться логи аудита. Вносим изменения в /etc/nginx/nginx.conf и 



```
[root@web ~]# grep nginx_conf /var/log/audit/audit.log

type=SYSCALL msg=audit(1656020159.342:1216): arch=c000003e syscall=188 success=yes exit=0 a0=cd47c0 a1=7f455b913f6a a2=ebb690 a3=24 items=1 ppid=24454 pid=24844 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=9 comm="vim" exe="/usr/bin/vim" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
type=SYSCALL msg=audit(1656020159.342:1217): arch=c000003e syscall=90 success=yes exit=0 a0=cd47c0 a1=81a4 a2=7ffca5037c70 a3=24 items=1 ppid=24454 pid=24844 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=9 comm="vim" exe="/usr/bin/vim" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
type=SYSCALL msg=audit(1656020159.342:1218): arch=c000003e syscall=188 success=yes exit=0 a0=cd47c0 a1=7f455b4c9e2f a2=eb68c0 a3=1c items=1 ppid=24454 pid=24844 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=9 comm="vim" exe="/usr/bin/vim" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
```
Далее настроим пересылку логов на удаленный сервер. Auditd по умолчанию не умеет пересылать логи, для пересылки на web-сервере потребуется установить пакет audispd-plugins: yum -y install audispd-plugins

Найдем и поменяем следующие строки в файле /etc/audit/auditd.conf:
```
log_format = RAW
name_format = HOSTNAME
```
В name_format указываем HOSTNAME, чтобы в логах на удаленном сервере отображалось имя хоста.

В файле /etc/audisp/plugins.d/au-remote.conf поменяем параметр active на yes:
```
active = yes
direction = out
path = /sbin/audisp-remote
type = always
#args =
format = string
```
В файле /etc/audisp/audisp-remote.conf требуется указать адрес сервера и порт, на который будут отправляться логи:
```
remote_server = 192.168.50.11
port = 60
```
Далее перезапускаем службу auditd: service auditd restart

На этом настройка web-сервера завершена. Далее настроим Log-сервер.

Отроем порт TCP 60, для этого уберем значки комментария в файле /etc/audit/auditd.conf:
```
tcp_listen_port = 60
```
Перезапускаем службу auditd: service auditd restart

На этом настройка пересылки логов аудита закончена. Можем попробовать поменять атрибут у файла /etc/nginx/nginx.conf и проверить на log-сервере, что пришла информация об изменении атрибута. Проверяем локально на web:
```
[root@web ~]# vim /etc/nginx/nginx.conf
[root@web ~]# ls -l /etc/nginx/nginx.conf
-rw-r--r--. 1 root root 2501 Jun 24 00:48 /etc/nginx/nginx.conf
[root@web ~]# chmod +x /etc/nginx/nginx.conf
[root@web ~]# ls -l /etc/nginx/nginx.conf
-rwxr-xr-x. 1 root root 2501 Jun 24 00:48 /etc/nginx/nginx.conf
```
Проверяем на сервере log:
```
[root@log ~]# grep web /var/log/audit/audit.log
...
node=web type=CWD msg=audit(1656022003.168:1264):  cwd="/root"
node=web type=PATH msg=audit(1656022003.168:1264): item=0 name="/etc/nginx/nginx.conf" inode=33638010 dev=08:01 mode=0100755 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
node=web type=PROCTITLE msg=audit(1656022003.168:1264): proctitle=63686D6F64002B78002F6574632F6E67696E782F6E67696E782E636F6E66
```
## Ansible и роль для сборщика логов (web+log)
Подготовим playbook для обеих машин. Разрушим тестовый машины командой vagran destroy и поднимем их заново (vagrant up) применив роль ansible.

Подключимся к каждой машине и проверим:
```
[root@log ~]# date
Wed Jun 29 01:11:41 MSK 2022

[root@web ~]# date
Wed Jun 29 01:11:44 MSK 2022
[root@web ~]# ss -tln | grep 80
LISTEN     0      128          *:80                       *:*
LISTEN     0      128       [::]:80                    [::]:*
[root@web ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2022-06-29 00:59:45 MSK; 12min ago
  Process: 4741 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 4739 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 4737 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
```
Дата впорядке. nginx работает

```
[root@log ~]# ss -tuln | grep 514
udp    UNCONN     0      0         *:514                   *:*
udp    UNCONN     0      0      [::]:514                [::]:*
tcp    LISTEN     0      25        *:514                   *:*
tcp    LISTEN     0      25     [::]:514                [::]:*
```
Открыты порты для rsyslog

```
[root@log ~]# cat /var/log/rsyslog/web/nginx_error.log
Jun 29 01:19:43 web nginx_error: 2022/06/29 01:19:43 [error] 5858#5858: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.50.1, server: _, request: "GET /favicon.ico HTTP/1.1", host: "192.168.50.10", referrer: "http://192.168.50.10/"

[root@log ~]# cat /var/log/rsyslog/web/nginx_access.log
Jun 29 01:19:42 web nginx_access: 192.168.50.1 - - [29/Jun/2022:01:19:42 +0300] "GET / HTTP/1.1" 200 4833 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:101.0) Gecko/20100101 Firefox/101.0"
Jun 29 01:19:43 web nginx_access: 192.168.50.1 - - [29/Jun/2022:01:19:43 +0300] "GET /favicon.ico HTTP/1.1" 404 3650 "http://192.168.50.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:101.0) Gecko/20100101 Firefox/101.0"
Jun 29 01:19:44 web nginx_access: 192.168.50.1 - - [29/Jun/2022:01:19:44 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:101.0) Gecko/20100101 Firefox/101.0"
```
При открытии страницы на web сервере в лог записывается информация об nginx


```
[root@log ~]# grep web /var/log/audit/audit.log
...
...
node=web type=SYSCALL msg=audit(1656455049.138:1835): arch=c000003e syscall=268 success=yes exit=0 a0=ffffffffffffff9c a1=1b730f0 a2=1ed a3=7ffef75c9e60 items=1 ppid=5785 pid=5860 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=6 comm="chmod" exe="/usr/bin/chmod" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
node=web type=CWD msg=audit(1656455049.138:1835):  cwd="/root"
node=web type=PATH msg=audit(1656455049.138:1835): item=0 name="/etc/nginx/nginx.conf" inode=33636920 dev=08:01 mode=0100644 ouid=1000 ogid=1000 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
node=web type=PROCTITLE msg=audit(1656455049.138:1835): proctitle=63686D6F64002B78002F6574632F6E67696E782F6E67696E782E636F6E66
```
При изменении прав на конфиг nginx (chmod +x /etc/nginx/nginx.conf) на лог сервер записывается информация


### Итоги/планы:
- Научиться передавать переменные в различные конфиги чтобы заполнять нужные значения (например ip адреса в конфиге)
- Объединить некоторые таски для их выполнения на нескольких машинах
- Работа со строками в сложных конфигах


