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

<h4>Создаём виртуальную машину.</h4>

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
</pre>

<p>Запустим систему:</p>

<pre>[user@localhost selinux]$ vagrant up</pre>

<p>Результатом выполнения команды vagrant up станет созданная виртуальная машина с установленным nginx, который работает на порту TCP 4881. Порт TCP 4881 уже проброшен до хоста. SELinux включен.</p>

<p>Во время развёртывания стенда попытка запустить nginx завершилась с ошибкой:</p>

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

<h4># 1 Запуск nginx на нестандартном порту 3-мя разными способами.</h4>

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

<h4>Способ 1. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью переключателей setsebool</h4>

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

<p>Копируем время "1656856482.995:803", в которое был записан этот лог, и с помощью утилиты audit2why смотрим информацию о запрете:</p>

<pre>[root@selinux ~]# grep 1656856482.995:803 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1656856482.995:803): avc:  denied  { name_bind } for  pid=2779 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

	Was caused by:
	The boolean nis_enabled was set incorrectly. 
	Description:
	Allow nis to enabled

	Allow access by executing:
	# setsebool -P nis_enabled 1
[root@selinux ~]#</pre>

<p>Утилита audit2why покажет почему трафик блокируется. Исходя из вывода утилиты, мы видим, что нам нужно поменять параметр nis_enabled. <br />
Включим параметр nis_enabled:</p>

<pre>[root@selinux ~]# setsebool -P nis_enabled on
[root@selinux ~]#</pre>

<p>и перезапустим nginx:</p>

<pre>[root@selinux ~]# systemctl restart nginx
[root@selinux ~]#</pre>

<p>Смотрим статус nginx:</p>

<pre>[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-07-03 15:02:57 UTC; 1min 6s ago
  Process: 21674 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21672 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21671 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21676 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21676 nginx: master process /usr/sbin/nginx
           └─21678 nginx: worker process

Jul 03 15:02:57 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 03 15:02:57 selinux nginx[21672]: nginx: the configuration file /etc/nginx/ngi...ok
Jul 03 15:02:57 selinux nginx[21672]: nginx: configuration file /etc/nginx/nginx.c...ul
Jul 03 15:02:57 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.
[root@selinux ~]#</pre>

<p>Также можно проверить работу nginx с помощью команды curl http://127.0.0.1:4881</p>

<pre>[root@selinux ~]# curl http://127.0.0.1:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css"> 

	html {
	background-image:url(img/html-background.png);
	background-color: white;
	font-family: "DejaVu Sans", "Liberation Sans", sans-serif;
	font-size: 0.85em;
	line-height: 1.25em;
	margin: 0 4% 0 4%;
	}

	body {
	border: 10px solid #fff;
	margin:0;
	padding:0;
	background: #fff;
	}

	/* Links */

	a:link { border-bottom: 1px dotted #ccc; text-decoration: none; color: #204d92; }
	a:hover { border-bottom:1px dotted #ccc; text-decoration: underline; color: green; }
	a:active {  border-bottom:1px dotted #ccc; text-decoration: underline; color: #204d92; }
	a:visited { border-bottom:1px dotted #ccc; text-decoration: none; color: #204d92; }
	a:visited:hover { border-bottom:1px dotted #ccc; text-decoration: underline; color: green; }
 
	.logo a:link,
	.logo a:hover,
	.logo a:visited { border-bottom: none; }

	.mainlinks a:link { border-bottom: 1px dotted #ddd; text-decoration: none; color: #eee; }
	.mainlinks a:hover { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }
	.mainlinks a:active { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }
	.mainlinks a:visited { border-bottom:1px dotted #ddd; text-decoration: none; color: white; }
	.mainlinks a:visited:hover { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }

	/* User interface styles */

	#header {
	margin:0;
	padding: 0.5em;
	background: #204D8C url(img/header-background.png);
	text-align: left;
	}

	.logo {
	padding: 0;
	/* For text only logo */
	font-size: 1.4em;
	line-height: 1em;
	font-weight: bold;
	}

	.logo img {
	vertical-align: middle;
	padding-right: 1em;
	}

	.logo a {
	color: #fff;
	text-decoration: none;
	}

	p {
	line-height:1.5em;
	}

	h1 { 
		margin-bottom: 0;
		line-height: 1.9em; }
	h2 { 
		margin-top: 0;
		line-height: 1.7em; }

	#content {
	clear:both;
	padding-left: 30px;
	padding-right: 30px;
	padding-bottom: 30px;
	border-bottom: 5px solid #eee;
	}

    .mainlinks {
        float: right;
        margin-top: 0.5em;
        text-align: right;
    }

    ul.mainlinks > li {
    border-right: 1px dotted #ddd;
    padding-right: 10px;
    padding-left: 10px;
    display: inline;
    list-style: none;
    }

    ul.mainlinks > li.last,
    ul.mainlinks > li.first {
    border-right: none;
    }

  </style>

</head>

<body>

<div id="header">

    <ul class="mainlinks">
        <li> <a href="http://www.centos.org/">Home</a> </li>
        <li> <a href="http://wiki.centos.org/">Wiki</a> </li>
        <li> <a href="http://wiki.centos.org/GettingHelp/ListInfo">Mailing Lists</a></li>
        <li> <a href="http://www.centos.org/download/mirrors/">Mirror List</a></li>
        <li> <a href="http://wiki.centos.org/irc">IRC</a></li>
        <li> <a href="https://www.centos.org/forums/">Forums</a></li>
        <li> <a href="http://bugs.centos.org/">Bugs</a> </li>
        <li class="last"> <a href="http://wiki.centos.org/Donate">Donate</a></li>
    </ul>

	<div class="logo">
		<a href="http://www.centos.org/"><img src="img/centos-logo.png" border="0"></a>
	</div>

</div>

<div id="content">

	<h1>Welcome to CentOS</h1>

	<h2>The Community ENTerprise Operating System</h2>

	<p><a href="http://www.centos.org/">CentOS</a> is an Enterprise-class Linux Distribution derived from sources freely provided to the public by Red Hat, Inc. for Red Hat Enterprise Linux.  CentOS conforms fully with the upstream vendors redistribution policy and aims to be functionally compatible. (CentOS mainly changes packages to remove upstream vendor branding and artwork.)</p>

	<p>CentOS is developed by a small but growing team of core developers.&nbsp; In turn the core developers are supported by an active user community including system administrators, network administrators, enterprise users, managers, core Linux contributors and Linux enthusiasts from around the world.</p>

	<p>CentOS has numerous advantages including: an active and growing user community, quickly rebuilt, tested, and QA'ed errata packages, an extensive <a href="http://www.centos.org/download/mirrors/">mirror network</a>, developers who are contactable and responsive, Special Interest Groups (<a href="http://wiki.centos.org/SpecialInterestGroup/">SIGs</a>) to add functionality to the core CentOS distribution, and multiple community support avenues including a <a href="http://wiki.centos.org/">wiki</a>, <a href="http://wiki.centos.org/irc">IRC Chat</a>, <a href="http://wiki.centos.org/GettingHelp/ListInfo">Email Lists</a>, <a href="https://www.centos.org/forums/">Forums</a>, <a href="http://bugs.centos.org/">Bugs Database</a>, and an <a href="http://wiki.centos.org/FAQ/">FAQ</a>.</p>

	</div>

</div>


</body>
</html>
[root@selinux ~]#</pre>

<p>Проверить статус параметра можно с помощью команды:</p>

<pre>[root@selinux ~]# getsebool -a | grep nis_enabled
nis_enabled --> on
[root@selinux ~]#</pre>

<p>В результате добились, что nginx работает на нестандартном порту 4881 с включенным selinux с помощью переключателя setsebool.</p>

<p>Вернём запрет работы nginx на порту 4881 обратно. Для этого отключим nis_enabled:</p>

<pre>[root@selinux ~]# setsebool -P nis_enabled off
[root@selinux ~]#</pre>

<p>После отключения nis_enabled служба nginx снова не запустится, убеждаемся:</p>

<pre>[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]#</pre>

<h4>Способ 2. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью добавления нестандартного порта в имеющийся тип</h4>

<p>Поиск имеющегося типа для http трафика:</p>

<pre>[root@selinux ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
[root@selinux ~]#</pre>

<p>Как видим, порт 4881 отсутствует в списке.</p>

<p>Добавим порт 4881 в тип http_port_t:</p>

<pre>[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]#</pre>

<p>Снова проверим:</p>

<pre>[root@selinux ~]# semanage port -a -t http_port_t -p tcp 4881
[root@selinux ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]#</pre>

<p>Порт 4881 добавлен в тип http_port_t</p>

<p>Теперь перезапустим службу nginx:</p>

<pre>[root@selinux ~]# systemctl restart nginx
[root@selinux ~]#</pre>

<p>и проверим её работу:</p>

<pre>[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-07-03 15:48:44 UTC; 23s ago
  Process: 21770 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21768 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21767 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21772 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21772 nginx: master process /usr/sbin/nginx
           └─21774 nginx: worker process

Jul 03 15:48:44 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 03 15:48:44 selinux nginx[21768]: nginx: the configuration file /etc/nginx/ngi...ok
Jul 03 15:48:44 selinux nginx[21768]: nginx: configuration file /etc/nginx/nginx.c...ul
Jul 03 15:48:44 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.
[root@selinux ~]#</pre>

<p>Также можно проверить работу nginx с помощью команды curl http://127.0.0.1:4881</p>

<pre>[root@selinux ~]# curl http://127.0.0.1:4881
<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01 Transitional//EN">
<html>
<head>
  <title>Welcome to CentOS</title>
  <style rel="stylesheet" type="text/css"> 

	html {
	background-image:url(img/html-background.png);
	background-color: white;
	font-family: "DejaVu Sans", "Liberation Sans", sans-serif;
	font-size: 0.85em;
	line-height: 1.25em;
	margin: 0 4% 0 4%;
	}

	body {
	border: 10px solid #fff;
	margin:0;
	padding:0;
	background: #fff;
	}

	/* Links */

	a:link { border-bottom: 1px dotted #ccc; text-decoration: none; color: #204d92; }
	a:hover { border-bottom:1px dotted #ccc; text-decoration: underline; color: green; }
	a:active {  border-bottom:1px dotted #ccc; text-decoration: underline; color: #204d92; }
	a:visited { border-bottom:1px dotted #ccc; text-decoration: none; color: #204d92; }
	a:visited:hover { border-bottom:1px dotted #ccc; text-decoration: underline; color: green; }
 
	.logo a:link,
	.logo a:hover,
	.logo a:visited { border-bottom: none; }

	.mainlinks a:link { border-bottom: 1px dotted #ddd; text-decoration: none; color: #eee; }
	.mainlinks a:hover { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }
	.mainlinks a:active { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }
	.mainlinks a:visited { border-bottom:1px dotted #ddd; text-decoration: none; color: white; }
	.mainlinks a:visited:hover { border-bottom:1px dotted #ddd; text-decoration: underline; color: white; }

	/* User interface styles */

	#header {
	margin:0;
	padding: 0.5em;
	background: #204D8C url(img/header-background.png);
	text-align: left;
	}

	.logo {
	padding: 0;
	/* For text only logo */
	font-size: 1.4em;
	line-height: 1em;
	font-weight: bold;
	}

	.logo img {
	vertical-align: middle;
	padding-right: 1em;
	}

	.logo a {
	color: #fff;
	text-decoration: none;
	}

	p {
	line-height:1.5em;
	}

	h1 { 
		margin-bottom: 0;
		line-height: 1.9em; }
	h2 { 
		margin-top: 0;
		line-height: 1.7em; }

	#content {
	clear:both;
	padding-left: 30px;
	padding-right: 30px;
	padding-bottom: 30px;
	border-bottom: 5px solid #eee;
	}

    .mainlinks {
        float: right;
        margin-top: 0.5em;
        text-align: right;
    }

    ul.mainlinks > li {
    border-right: 1px dotted #ddd;
    padding-right: 10px;
    padding-left: 10px;
    display: inline;
    list-style: none;
    }

    ul.mainlinks > li.last,
    ul.mainlinks > li.first {
    border-right: none;
    }

  </style>

</head>

<body>

<div id="header">

    <ul class="mainlinks">
        <li> <a href="http://www.centos.org/">Home</a> </li>
        <li> <a href="http://wiki.centos.org/">Wiki</a> </li>
        <li> <a href="http://wiki.centos.org/GettingHelp/ListInfo">Mailing Lists</a></li>
        <li> <a href="http://www.centos.org/download/mirrors/">Mirror List</a></li>
        <li> <a href="http://wiki.centos.org/irc">IRC</a></li>
        <li> <a href="https://www.centos.org/forums/">Forums</a></li>
        <li> <a href="http://bugs.centos.org/">Bugs</a> </li>
        <li class="last"> <a href="http://wiki.centos.org/Donate">Donate</a></li>
    </ul>

	<div class="logo">
		<a href="http://www.centos.org/"><img src="img/centos-logo.png" border="0"></a>
	</div>

</div>

<div id="content">

	<h1>Welcome to CentOS</h1>

	<h2>The Community ENTerprise Operating System</h2>

	<p><a href="http://www.centos.org/">CentOS</a> is an Enterprise-class Linux Distribution derived from sources freely provided to the public by Red Hat, Inc. for Red Hat Enterprise Linux.  CentOS conforms fully with the upstream vendors redistribution policy and aims to be functionally compatible. (CentOS mainly changes packages to remove upstream vendor branding and artwork.)</p>

	<p>CentOS is developed by a small but growing team of core developers.&nbsp; In turn the core developers are supported by an active user community including system administrators, network administrators, enterprise users, managers, core Linux contributors and Linux enthusiasts from around the world.</p>

	<p>CentOS has numerous advantages including: an active and growing user community, quickly rebuilt, tested, and QA'ed errata packages, an extensive <a href="http://www.centos.org/download/mirrors/">mirror network</a>, developers who are contactable and responsive, Special Interest Groups (<a href="http://wiki.centos.org/SpecialInterestGroup/">SIGs</a>) to add functionality to the core CentOS distribution, and multiple community support avenues including a <a href="http://wiki.centos.org/">wiki</a>, <a href="http://wiki.centos.org/irc">IRC Chat</a>, <a href="http://wiki.centos.org/GettingHelp/ListInfo">Email Lists</a>, <a href="https://www.centos.org/forums/">Forums</a>, <a href="http://bugs.centos.org/">Bugs Database</a>, and an <a href="http://wiki.centos.org/FAQ/">FAQ</a>.</p>

	</div>

</div>


</body>
</html>
[root@selinux ~]#</pre>

<p>В результате добились, что nginx работает на нестандартном порту 4881 с включенным selinux c помощью добавления нестандартного порта в имеющийся тип.</p>

<p>Удалим нестандартный порт из имеющегося типа можно с помощью команды:</p>

<pre>[root@selinux ~]# semanage port -d -t http_port_t -p tcp 4881
[root@selinux ~]#</pre>

<p>Проверяем, присутствует ли порт 4881 в списке http_port_t:</p>

<pre>[root@selinux ~]# semanage port -l | grep http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
[root@selinux ~]#</pre>

<p>Пробуем перезапустить nginx:</p>

<pre>[root@selinux ~]# systemctl restart nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]#</pre>

<p>Смотрим статус работы nginx:</p>

<pre>[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sun 2022-07-03 16:03:00 UTC; 10min ago
  Process: 21770 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21805 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 21804 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21772 (code=exited, status=0/SUCCESS)

Jul 03 16:03:00 selinux systemd[1]: Stopped The nginx HTTP and reverse proxy server.
Jul 03 16:03:00 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 03 16:03:00 selinux nginx[21805]: nginx: the configuration file /etc/nginx/ngi...ok
Jul 03 16:03:00 selinux nginx[21805]: nginx: [emerg] bind() to 0.0.0.0:4881 failed...d)
Jul 03 16:03:00 selinux nginx[21805]: nginx: configuration file /etc/nginx/nginx.c...ed
Jul 03 16:03:00 selinux systemd[1]: nginx.service: control process exited, code=e...s=1
Jul 03 16:03:00 selinux systemd[1]: Failed to start The nginx HTTP and reverse pr...er.
Jul 03 16:03:00 selinux systemd[1]: Unit nginx.service entered failed state.
Jul 03 16:03:00 selinux systemd[1]: nginx.service failed.
Hint: Some lines were ellipsized, use -l to show in full.
[root@selinux ~]#</pre>

<h4>Способ 3. Разрешим в SELinux работу nginx на порту TCP 4881 c помощью формирования и установки модуля SELinux</h4>

<p>Попробуем снова запустить nginx:</p>

<pre>[root@selinux ~]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code. See "systemctl status nginx.service" and "journalctl -xe" for details.
[root@selinux ~]#</pre>

<p>Nginx не запускается, так как SELinux продолжает его блокировать. Посмотрим логи SELinux, которые относятся к nginx:</p>

<pre>[root@selinux ~]# grep nginx /var/log/audit/audit.log
...
type=SYSCALL msg=audit(1656864180.643:895): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=557c34f92858 a2=10 a3=7ffc33ea7930 items=0 ppid=1 pid=21805 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1656864180.643:896): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
type=AVC msg=audit(1656865088.306:897): avc:  denied  { name_bind } for  pid=21822 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
type=SYSCALL msg=audit(1656865088.306:897): arch=c000003e syscall=49 success=no exit=-13 a0=6 a1=5590f606b858 a2=10 a3=7ffe850b7890 items=0 ppid=1 pid=21822 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nginx" exe="/usr/sbin/nginx" subj=system_u:system_r:httpd_t:s0 key=(null)
type=SERVICE_START msg=audit(1656865088.306:898): pid=1 uid=0 auid=4294967295 ses=4294967295 subj=system_u:system_r:init_t:s0 msg='unit=nginx comm="systemd" exe="/usr/lib/systemd/systemd" hostname=? addr=? terminal=? res=failed'
[root@selinux ~]#</pre>

<p>Воспользуемся утилитой audit2allow для того, чтобы на основе логов SELinux сделать модуль, разрешающий работу nginx на нестандартном порту:</p>

<pre>[root@selinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@selinux ~]#</pre>

<p>Audit2allow сформировал модуль и сообщил нам команду, с помощью которой можно применить данный модуль:</p>

<pre>[root@selinux ~]# semodule -i nginx.pp
[root@selinux ~]#</pre>

<p>Попробуем снова запустить nginx:</p>

<pre>[root@selinux ~]# systemctl start nginx
[root@selinux ~]#</pre>

<p>Смотрим статус запущенной службы nginx:</p>

<pre>[root@selinux ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2022-07-03 17:19:42 UTC; 1min 12s ago
  Process: 21869 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 21866 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 21865 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 21871 (nginx)
   CGroup: /system.slice/nginx.service
           ├─21871 nginx: master process /usr/sbin/nginx
           └─21873 nginx: worker process

Jul 03 17:19:42 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jul 03 17:19:42 selinux nginx[21866]: nginx: the configuration file /etc/nginx/ngi...ok
Jul 03 17:19:42 selinux nginx[21866]: nginx: configuration file /etc/nginx/nginx.c...ul
Jul 03 17:19:42 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
Hint: Some lines were ellipsized, use -l to show in full.
[root@selinux ~]#</pre>

<p>Как видим, после добавления модуля nginx запустился без ошибок. При использовании модуля изменения сохранятся после перезагрузки.</p>

<p>Просмотр всех установленных модулей: semodule -l, так как список будет внушительным, поэтому сделаем отбор по nginx:</p>

<pre>[root@selinux ~]# semodule -l | grep nginx
nginx	1.0
[root@selinux ~]#</pre>

<p>В результате добились, что nginx работает на нестандартном порту 4881 с включенным selinux за счет формирования и установки модуля SELinux.</p>

<p>Для удаления модуля воспользуемся командой:</p>

<pre>[root@selinux ~]# semodule -r nginx
libsemanage.semanage_direct_remove_key: Removing last nginx module (no other nginx module exists at another priority).
[root@selinux ~]#</pre>

<p>Снова преверим наличие nginx в списке установленных модулей:</p>

<pre>[root@selinux ~]# semodule -l | grep nginx
[root@selinux ~]#</pre>

<p>Как видим, nginx отсутствует в списке установленных модулей.</p>

<h4># 2. Обеспечение работоспособности приложения при включенном SELinux</h4>

<p>Для того, чтобы развернуть стенд потребуется хост, с установленным git и ansible.</p>

<p>Инструкция по установке Ansible -<br />
https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html<br />
Инструкция по установке Git -<br />
https://git-scm.com/book/ru/v2/%D0%92%D0%B2%D0%B5%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5-%D0%A3%D1%81%D1%82%D0%B0%D0%BD%D0%BE%D0%B2%D0%BA%D0%B0-Git</p>

<p>Выполним клонирование репозитория:</p>

<pre>[user@localhost otus]$ git clone https://github.com/mbfx/otus-linux-adm.git
Cloning into 'otus-linux-adm'...
remote: Enumerating objects: 542, done.
remote: Counting objects: 100% (440/440), done.
remote: Compressing objects: 100% (295/295), done.
remote: Total 542 (delta 118), reused 381 (delta 69), pack-reused 102
Receiving objects: 100% (542/542), 1.38 MiB | 0 bytes/s, done.
Resolving deltas: 100% (133/133), done.
[user@localhost otus]$</pre>

<p>Перейдём в каталог со стендом:</p>

<pre>[user@localhost otus]$ cd ./otus-linux-adm/selinux_dns_problems/
[user@localhost selinux_dns_problems]$</pre>

<p>Развернём 2 ВМ с помощью vagrant:</p>

<pre>[user@localhost selinux_dns_problems]$ vagrant up</pre>

<p>Проверим статусы ВМ развернутого стенда:</p>

<pre>[user@localhost selinux_dns_problems]$ vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost selinux_dns_problems]$</pre>

<p>С помощью SSH подключимся к клиенту:</p>

<pre>[user@localhost selinux_dns_problems]$ vagrant ssh client
Last login: Sun Jul  3 19:57:26 2022 from 10.0.2.2
###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################
[vagrant@client ~]$</pre>

<p>Попробуем внести изменения в зону:</p>

<pre>[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key 
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
[vagrant@client ~]$</pre>

<p>Как видим, внести изменения не получилось. Посмотрим логи SELinux, чтобы понять в чём может быть проблема. <br />
Для этого воспользуемся утилитой audit2why (предварительно зайдем под правами root):</p>

<pre>[vagrant@client ~]$ sudo -i
[root@client ~]# cat /var/log/audit/audit.log | audit2why
[root@client ~]#</pre>

<p>Тут мы видим, что на клиенте отсутствуют ошибки.</p>

<p>Не закрывая сессию на клиенте, подключимся к серверу ns01 и проверим логи SELinux:</p>

<pre>[user@localhost selinux_dns_problems]$ vagrant ssh ns01
Last login: Sun Jul  3 19:55:29 2022 from 10.0.2.2
[vagrant@ns01 ~]$</pre>

<p>Заходим под правами root:</p>

<pre>[vagrant@ns01 ~]$ sudo -i
[root@ns01 ~]#</pre>

<p>Также смотрим логи SELinux:</p>

<pre>[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1656878864.895:1898): avc:  denied  { create } for  pid=5066 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

	Was caused by:
		Missing type enforcement (TE) allow rule.

		You can use audit2allow to generate a loadable module to allow this access.

[root@ns01 ~]#</pre>

<p>В логах мы видим, что ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t. <br />
Проверим данную проблему в каталоге /etc/named:</p>

<pre>[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
[root@ns01 ~]#</pre>

<p>Тут мы также видим, что контекст безопасности неправильный. Проблема заключается в том, что конфигурационные файлы лежат в другом каталоге. <br />Посмотреть в каком каталоги должны лежать, файлы, чтобы на них распространялись правильные политики SELinux можно с помощью команды:</p>

<pre>[root@ns01 ~]# semanage fcontext -l | grep named
/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/etc/unbound(/.*)?                                 all files          system_u:object_r:named_conf_t:s0 
/var/run/bind(/.*)?                                all files          system_u:object_r:named_var_run_t:s0 
/var/log/named.*                                   regular file       system_u:object_r:named_log_t:s0 
/var/run/named(/.*)?                               all files          system_u:object_r:named_var_run_t:s0 
/var/named/data(/.*)?                              all files          system_u:object_r:named_cache_t:s0 
/dev/xen/tapctrl.*                                 named pipe         system_u:object_r:xenctl_t:s0 
/var/run/unbound(/.*)?                             all files          system_u:object_r:named_var_run_t:s0 
/var/lib/softhsm(/.*)?                             all files          system_u:object_r:named_cache_t:s0 
/var/lib/unbound(/.*)?                             all files          system_u:object_r:named_cache_t:s0 
/var/named/slaves(/.*)?                            all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot(/.*)?                            all files          system_u:object_r:named_conf_t:s0 
/etc/named\.rfc1912.zones                          regular file       system_u:object_r:named_conf_t:s0 
/var/named/dynamic(/.*)?                           all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/etc(/.*)?                        all files          system_u:object_r:etc_t:s0 
/var/named/chroot/lib(/.*)?                        all files          system_u:object_r:lib_t:s0 
/var/named/chroot/proc(/.*)?                       all files          <<None>>
/var/named/chroot/var/tmp(/.*)?                    all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/usr/lib(/.*)?                    all files          system_u:object_r:lib_t:s0 
/var/named/chroot/etc/pki(/.*)?                    all files          system_u:object_r:cert_t:s0 
/var/named/chroot/run/named.*                      all files          system_u:object_r:named_var_run_t:s0 
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0 
/usr/lib/systemd/system/named.*                    regular file       system_u:object_r:named_unit_file_t:s0 
/var/named/chroot/var/run/dbus(/.*)?               all files          system_u:object_r:system_dbusd_var_run_t:s0 
/usr/lib/systemd/system/unbound.*                  regular file       system_u:object_r:named_unit_file_t:s0 
/var/named/chroot/var/log/named.*                  regular file       system_u:object_r:named_log_t:s0 
/var/named/chroot/var/run/named.*                  all files          system_u:object_r:named_var_run_t:s0 
/var/named/chroot/var/named/data(/.*)?             all files          system_u:object_r:named_cache_t:s0 
/usr/lib/systemd/system/named-sdb.*                regular file       system_u:object_r:named_unit_file_t:s0 
/var/named/chroot/var/named/slaves(/.*)?           all files          system_u:object_r:named_cache_t:s0 
/var/named/chroot/etc/named\.rfc1912.zones         regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/var/named/dynamic(/.*)?          all files          system_u:object_r:named_cache_t:s0 
/var/run/ndc                                       socket             system_u:object_r:named_var_run_t:s0 
/dev/gpmdata                                       named pipe         system_u:object_r:gpmctl_t:s0 
/dev/initctl                                       named pipe         system_u:object_r:initctl_t:s0 
/dev/xconsole                                      named pipe         system_u:object_r:xconsole_device_t:s0 
/usr/sbin/named                                    regular file       system_u:object_r:named_exec_t:s0 
/etc/named\.conf                                   regular file       system_u:object_r:named_conf_t:s0 
/usr/sbin/lwresd                                   regular file       system_u:object_r:named_exec_t:s0 
/var/run/initctl                                   named pipe         system_u:object_r:initctl_t:s0 
/usr/sbin/unbound                                  regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/named-sdb                                regular file       system_u:object_r:named_exec_t:s0 
/var/named/named\.ca                               regular file       system_u:object_r:named_conf_t:s0 
/etc/named\.root\.hints                            regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/dev                              directory          system_u:object_r:device_t:s0 
/etc/rc\.d/init\.d/named                           regular file       system_u:object_r:named_initrc_exec_t:s0 
/usr/sbin/named-pkcs11                             regular file       system_u:object_r:named_exec_t:s0 
/etc/rc\.d/init\.d/unbound                         regular file       system_u:object_r:named_initrc_exec_t:s0 
/usr/sbin/unbound-anchor                           regular file       system_u:object_r:named_exec_t:s0 
/usr/sbin/named-checkconf                          regular file       system_u:object_r:named_checkconf_exec_t:s0 
/usr/sbin/unbound-control                          regular file       system_u:object_r:named_exec_t:s0 
/var/named/chroot_sdb/dev                          directory          system_u:object_r:device_t:s0 
/var/named/chroot/var/log                          directory          system_u:object_r:var_log_t:s0 
/var/named/chroot/dev/log                          socket             system_u:object_r:devlog_t:s0 
/etc/rc\.d/init\.d/named-sdb                       regular file       system_u:object_r:named_initrc_exec_t:s0 
/var/named/chroot/dev/null                         character device   system_u:object_r:null_device_t:s0 
/var/named/chroot/dev/zero                         character device   system_u:object_r:zero_device_t:s0 
/usr/sbin/unbound-checkconf                        regular file       system_u:object_r:named_exec_t:s0 
/var/named/chroot/dev/random                       character device   system_u:object_r:random_device_t:s0 
/var/run/systemd/initctl/fifo                      named pipe         system_u:object_r:initctl_t:s0 
/var/named/chroot/etc/rndc\.key                    regular file       system_u:object_r:dnssec_t:s0 
/usr/share/munin/plugins/named                     regular file       system_u:object_r:services_munin_plugin_exec_t:s0 
/var/named/chroot_sdb/dev/null                     character device   system_u:object_r:null_device_t:s0 
/var/named/chroot_sdb/dev/zero                     character device   system_u:object_r:zero_device_t:s0 
/var/named/chroot/etc/localtime                    regular file       system_u:object_r:locale_t:s0 
/var/named/chroot/etc/named\.conf                  regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot_sdb/dev/random                   character device   system_u:object_r:random_device_t:s0 
/etc/named\.caching-nameserver\.conf               regular file       system_u:object_r:named_conf_t:s0 
/usr/lib/systemd/systemd-hostnamed                 regular file       system_u:object_r:systemd_hostnamed_exec_t:s0 
/var/named/chroot/var/named/named\.ca              regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/etc/named\.root\.hints           regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/etc/named\.caching-nameserver\.conf regular file       system_u:object_r:named_conf_t:s0 
/var/named/chroot/lib64 = /usr/lib
/var/named/chroot/usr/lib64 = /usr/lib
[root@ns01 ~]#</pre>

<p>Изменим тип контекста безопасности для каталога /etc/named:</p>

<pre>[root@ns01 ~]# chcon -R -t named_zone_t /etc/named
[root@ns01 ~]#</pre>

<p>Снова проверим каталог /etc/named:</p>

<pre>[root@ns01 ~]# ls -laZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
[root@ns01 ~]#</pre>

<p>Попробуем снова внести изменения с клиента:</p>

<pre>[vagrant@client ~]$ nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15                                   
> send
> quit
[vagrant@client ~]$</pre>

<p>Получилось внести изменения. Для проверки сделаем запрос с помощью dig:</p>

<pre>[vagrant@client ~]$ dig www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> www.ddns.lab
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 45783
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 2 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sun Jul 03 20:48:59 UTC 2022
;; MSG SIZE  rcvd: 96

[vagrant@client ~]$</pre>

<p>Видим, что изменения применились. Перезагрузим хосты:</p>

<pre>[root@ns01 ~]# shutdown -r now
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
[user@localhost selinux_dns_problems]$ vagrant ssh ns01
Last login: Sun Jul  3 20:18:53 2022 from 10.0.2.2
[vagrant@ns01 ~]$</pre>

<pre>[vagrant@client ~]$ sudo shutdown -r now
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
[user@localhost selinux_dns_problems]$ vagrant ssh client
Last login: Sun Jul  3 20:04:19 2022 from 10.0.2.2
###############################
### Welcome to the DNS lab! ###
###############################

- Use this client to test the enviroment
- with dig or nslookup. Ex:
    dig @192.168.50.10 ns01.dns.lab

- nsupdate is available in the ddns.lab zone. Ex:
    nsupdate -k /etc/named.zonetransfer.key
    server 192.168.50.10
    zone ddns.lab 
    update add www.ddns.lab. 60 A 192.168.50.15
    send

- rndc is also available to manage the servers
    rndc -c ~/rndc.conf reload

###############################
### Enjoy! ####################
###############################
[vagrant@client ~]$</pre>

<p>и ещё раз сделаем запрос с помощью dig:</p>

<pre>[vagrant@client ~]$ dig @192.168.50.10 www.ddns.lab

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.9 <<>> @192.168.50.10 www.ddns.lab
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 64585
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.ddns.lab.			IN	A

;; ANSWER SECTION:
www.ddns.lab.		60	IN	A	192.168.50.15

;; AUTHORITY SECTION:
ddns.lab.		3600	IN	NS	ns01.dns.lab.

;; ADDITIONAL SECTION:
ns01.dns.lab.		3600	IN	A	192.168.50.10

;; Query time: 1 msec
;; SERVER: 192.168.50.10#53(192.168.50.10)
;; WHEN: Sun Jul 03 20:58:04 UTC 2022
;; MSG SIZE  rcvd: 96

[vagrant@client ~]$</pre>

<p>Как видим, после перезагрузки настройки сохранились.<br />
Для того, чтобы вернуть правила обратно, на стороне сервера вводим команду:</p>

<pre>[vagrant@ns01 ~]$ sudo restorecon -v -R /etc/named
restorecon reset /etc/named context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.dns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic context unconfined_u:object_r:named_zone_t:s0->unconfined_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1 context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/dynamic/named.ddns.lab.view1.jnl context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
restorecon reset /etc/named/named.50.168.192.rev context system_u:object_r:named_zone_t:s0->system_u:object_r:etc_t:s0
[vagrant@ns01 ~]$</pre>
