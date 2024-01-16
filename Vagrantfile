# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :"otus-task12" => {
		:box_name => "centos/7",
		:box_version => "2004.01",
  },
}

Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
	  config.vm.synced_folder ".", "/vagrant", disabled: true
	  config.vm.define boxname do |box|
		box.vm.box = boxconfig[:box_name]
		box.vm.box_version = boxconfig[:box_version]
		box.vm.host_name = "otus-task12"
		box.vm.network "forwarded_port", guest: 4881, host: 4881
		box.vm.provider :virtualbox do |vb|
			  vb.customize ["modifyvm", :id, "--memory", "1024"]
			  needsController = false
		end

		box.vm.provision "shell", inline: <<-SHELL
		  #Установить epel-release
		  yum install -y epel-release
		  #Установить nginx, утилиты для работы SELinux и telnet
		  yum install -y nginx
		  yum install policycoreutils-python telnet
		  #Поменять порт для nginx
		  sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
		  sed -i 's/listen	   80;/listen	   4881;/' /etc/nginx/nginx.conf
		  # Попытаться запустить nginx
		  systemctl start nginx
		  systemctl status nginx
		SHELL
	end
  end
end
