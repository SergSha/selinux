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

<p>Подключимся к ней с помощью SSH:</p>

<pre>[user@localhost selinux]$ vagrant ssh
[vagrant@selinux ~]$</pre>

<p>Заходим под правами root:</p>

<pre>[vagrant@selinux ~]$ sudo -i
[root@selinux ~]#</pre>

<p>Для начала создаём файл с конфигурацией для сервиса в директории /etc/sysconfig - из неё сервис будет брать необходимые переменные:</p>

<pre>[root@selinux ~]# vi /etc/sysconfig/watchlog</pre>

<pre># Configuration file for my watchlog service
# Place it to /etc/sysconfig

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log</pre>

<p>Затем создаем /var/log/watchlog.log и пишем туда строки на своё усмотрение,
плюс ключевое слово 'ALERT'</p>

<pre>[root@selinux ~]# vi /var/log/watchlog.log</pre>
