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

<h4># 1. Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/sysconfig).</h4>

<p>В домашней директории создадим директорию systemd, в которой будут храниться настройки виртуальной машины:</p>

<pre>[user@localhost otus]$ mkdir ./systemd
[user@localhost otus]$</pre>

<p>Перейдём в директорию systemd:</p>

<pre>[user@localhost otus]$ cd ./systemd/
[user@localhost systemd]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost systemd]$ vi ./Vagrantfile</pre>

<p>Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
# vim: set ft=ruby :
home = ENV['HOME']
ENV["LC_ALL"] = "en_US.UTF-8"

MACHINES = {
  :systemd => {
        :box_name => "centos/7",
        :box_version => "1804.02",
        :ip_addr => '192.168.56.101',
    :disks => {
        :sata1 => {
            :dfile => home + '/VirtualBox VMs/sata1.vdi',
            :size => 10240,
            :port => 1
        },
        :sata2 => {
            :dfile => home + '/VirtualBox VMs/sata2.vdi',
            :size => 2048, # Megabytes
            :port => 2
        },
        :sata3 => {
            :dfile => home + '/VirtualBox VMs/sata3.vdi',
            :size => 1024, # Megabytes
            :port => 3
        },
        :sata4 => {
            :dfile => home + '/VirtualBox VMs/sata4.vdi',
            :size => 1024,
            :port => 4
        }
    }
  },
}

Vagrant.configure("2") do |config|

    config.vm.box_version = "1804.02"
    MACHINES.each do |boxname, boxconfig|
  
        config.vm.define boxname do |box|
  
            box.vm.box = boxconfig[:box_name]
            box.vm.host_name = boxname.to_s
  
            #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset
  
            box.vm.network "private_network", ip: boxconfig[:ip_addr]
  
            box.vm.provider :virtualbox do |vb|
                    vb.customize ["modifyvm", :id, "--memory", "256"]
                    needsController = false
            boxconfig[:disks].each do |dname, dconf|
                unless File.exist?(dconf[:dfile])
                  vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                                  needsController =  true
                            end
  
            end
                    if needsController == true
                       vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                       boxconfig[:disks].each do |dname, dconf|
                           vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                       end
                    end
            end
  
        box.vm.provision "shell", inline: <<-SHELL
            mkdir -p ~root/.ssh
            cp ~vagrant/.ssh/auth* ~root/.ssh
            yum install -y mdadm smartmontools hdparm gdisk
          SHELL
  
        end
    end
  end
</pre>

<p>Запустим систему:</p>

<pre>[user@localhost systemd]$ vagrant up</pre>

<p>и войдём в неё:</p>

<pre>[user@localhost systemd]$ vagrant ssh
[vagrant@systemd ~]$</pre>

<p>Заходим под правами root:</p>

<pre>[vagrant@systemd ~]$ sudo -i
[root@systemd ~]#</pre>

<p>Для начала создаём файл с конфигурацией для сервиса в директории /etc/sysconfig - из неё сервис будет брать необходимые переменные:</p>

<pre>[root@systemd ~]# vi /etc/sysconfig/watchlog</pre>

<pre># Configuration file for my watchlog service
# Place it to /etc/sysconfig

# File and word in that file that we will be monit
WORD="ALERT"
LOG=/var/log/watchlog.log</pre>

<p>Затем создаем /var/log/watchlog.log и пишем туда строки на своё усмотрение,
плюс ключевое слово 'ALERT'</p>

<pre>[root@systemd ~]# vi /var/log/watchlog.log</pre>
