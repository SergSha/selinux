<h3>### SELinux ###</h3>

<h4>Цель:</h4>
<p>Тренируем умение работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.</p>

<h4>Описание домашнего задания</h4>
<ol>
  <li>Запустить nginx на нестандартном порту 3-мя разными способами:
  <ul>
    <li>переключатели setsebool;</li>
    <li>добавление нестандартного порта в имеющийся тип;</li>
    <li>формирование и установка модуля SELinux. К сдаче:
    <ul>
      <li>README с описанием каждого решения (скриншоты и демонстрация приветствуются).</li>
    </ul></li>
  </ul></li>
  <li>Обеспечить работоспособность приложения при включенном selinux.
  <ul>
    <li>развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;</li>
    <li>выяснить причину неработоспособности механизма обновления зоны (см. README);</li>
    <li>предложить решение (или решения) для данной проблемы;</li>
    <li>выбрать одно из решений для реализации, предварительно обосновав выбор;</li>
    <li>реализовать выбранное решение и продемонстрировать его работоспособность. К сдаче:
    <ul>
      <li>README с анализом причины неработоспособности, возможными способами решения и обоснованием выбора одного из них;</li>
      <li>исправленный стенд или демонстрация работоспособной системы скриншотами и описанием.</li>
    </ul></li>
  </ul></li>
</ol>

<h4>Критерии оценки:</h4>
<p>Статус "Принято" ставится при выполнении следующих условий:</p>
<ul>
  <li>для задания 1 описаны, реализованы и продемонстрированы все 3 способа решения;</li>
  <li>для задания 2 описана причина неработоспособности механизма обновления зоны;</li>
  <li>для задания 2 реализован и продемонстрирован один из способов решения. Опционально для выполнения:</li>
  <li>для задания 2 предложено более одного способа решения;</li>
  <li>для задания 2 обоснованно(!) выбран один из способов решения.</li>
</ul>

<h4># 1. Создаём виртуальную машину.</h4>

<p>В домашней директории создадим директорию selinux, в которой будут храниться настройки виртуальной машины:</p>

<pre>[user@localhost otus]$ mkdir ./selinux
[user@localhost otus]$</pre>

<p>Перейдём в директорию selinux:</p>

<pre>[user@localhost otus]$ cd ./selinux/
[user@localhost selinux]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost selinux]$ vi ./Vagrantfile</pre>

<p>Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :selinux => {
    :box_name => "centos/7",
    :box_version => "2004.01",
    #:provision => "test.sh",
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
        yum install -y epel-release
        #install nginx
        yum install -y nginx
        #install policycoreutils-python
        yum install -y policycoreutils-python
        #change nginx port
        sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
        sed -i 's/listen  80;/listen  4881;/' /etc/nginx/nginx.conf
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
</pre>

<p>Запустим систему:</p>

<pre>[user@localhost selinux]$ vagrant up</pre>

<p>Результатом выполнения команды vagrant up станет созданная виртуальная машина с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.</p>

<p>Во время развёртывания стенда попытка запустить nginx завершится с ошибкой:</p>

<pre>...
selinux: ● nginx.service - The nginx HTTP and reverse proxy server
    selinux:    Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
    selinux:    Active: failed (Result: exit-code) since Sun 2022-07-03 13:54:43 UTC; 13ms ago
    selinux:   Process: 2779 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
    selinux:   Process: 2778 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    selinux: 
    selinux: Jul 03 13:54:42 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
    selinux: Jul 03 13:54:43 selinux nginx[2779]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    selinux: Jul 03 13:54:43 selinux nginx[2779]: nginx: [emerg] bind() to [::]:4881 failed (13: Permission denied)
    selinux: Jul 03 13:54:43 selinux nginx[2779]: nginx: configuration file /etc/nginx/nginx.conf test failed
    selinux: Jul 03 13:54:43 selinux systemd[1]: nginx.service: control process exited, code=exited status=1
    selinux: Jul 03 13:54:43 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
    selinux: Jul 03 13:54:43 selinux systemd[1]: Unit nginx.service entered failed state.
    selinux: Jul 03 13:54:43 selinux systemd[1]: nginx.service failed.
...</pre>

<p>Данная ошибка появляется из-за того, что SELinux блокирует работу nginx на нестандартном порту.</p>

<p>Подключимся к ней с помощью SSH:</p>

<pre>[user@localhost selinux]$ vagrant ssh
[vagrant@selinux ~]$</pre>

<p>Заходим под правами root:</p>

<pre>[vagrant@selinux ~]$ sudo -i
[root@selinux ~]#</pre>

<h4># 2 Запуск nginx на нестандартном порту 3-мя разными способами.</h4>

<p>Для начала проверим, что в ОС отключен файервол:</p>

<pre>[root@selinux ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)
[root@selinux ~]#</pre>

<p>Также можно проверить, что конфигурация nginx настроена без ошибок:</p>

<pre>[root@selinux ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@selinux ~]#</pre>

<p>Далее проверим режим работы SELinux:</p>

<pre>[root@selinux ~]# getenforce
Enforcing
[root@selinux ~]#</pre>

<p>Как видим, у нас режим Enforcing, а это означает, что SELinux будет блокировать запрещенную активность.</p>

<h4>Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool</h4>

<p>Ищем в логах (/var/log/audit/audit.log) информацию о блокировании порта:</p>

<pre>[root@selinux ~]# less /var/log/audit/audit.log</pre>

<pre>...
type=AVC msg=audit(1656856482.995:803): avc:  denied  { name_bind } for  pid=2779 comm=
"nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unres
erved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1656856482.995:803): arch=c000003e syscall=49 success=no exit=-1
3 a0=7 a1=558a2b7d08a8 a2=1c a3=7ffda9700174 items=0 ppid=1 pid=2779 auid=4294967295 ui
d=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="n
ginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=PROCTITLE msg=audit(1656856482.995:803): proctitle=2F7573722F7362696E2F6E67696E780
02D74
type=SERVICE_START msg=audit(1656856482.995:804): pid=1 uid=0 auid=4294967295 ses=42949
67295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/sys
temd/systemd" hostname=? addr=? terminal=? res=failed'
...</pre>

<p>Копируем время "1656856482.995:803", в которое был записан этот лог, и с помощью утилиты audit2why смотрим информации о запрете:</p>

<pre>[root@selinux ~]# grep 1656856482.995:803 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1656856482.995:803): avc:  denied  { name_bind } for  pid=2779 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
[root@selinux ~]#</pre>





