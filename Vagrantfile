# -*- mode: ruby -*-
# vi: set ft=ruby :

# Require YAML module
require 'yaml'

config = YAML.load_file(File.join(File.dirname(__FILE__), 'config.yml'))

base_box=config['environment']['base_box']

master_ip=config['environment']['masterip']

domain=config['environment']['domain']

boxes = config['boxes']

boxes_hostsfile_entries=""

 boxes.each do |box|
   boxes_hostsfile_entries=boxes_hostsfile_entries+box['mgmt_ip'] + ' ' +  box['name'] + ' ' + box['name']+'.'+domain+'\n'
 end

#puts boxes_hostsfile_entries

update_hosts = <<SCRIPT
    echo "127.0.0.1 localhost" >/etc/hosts
    echo -e "#{boxes_hostsfile_entries}" |tee -a /etc/hosts
SCRIPT

Vagrant.configure(2) do |config|
  config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"        
  config.ssh.forward_x11
  config.vm.box = base_box
  config.vm.synced_folder "tmp_deploying_stage/", "/tmp_deploying_stage",create:true
  
  config.vm.synced_folder "nginx_keys/", "/nginx_keys"
  
  config.vm.synced_folder "src/", "/src",create:true
  boxes.each do |node|
    config.vm.define node['name'] do |config|
      config.vm.hostname = node['name']
      config.vm.provider "virtualbox" do |v|
        v.name = node['name']
        v.customize ["modifyvm", :id, "--memory", node['mem']]
        v.customize ["modifyvm", :id, "--cpus", node['cpu']]

        v.customize ["modifyvm", :id, "--nictype1", "Am79C973"]
        v.customize ["modifyvm", :id, "--nictype2", "Am79C973"]
        v.customize ["modifyvm", :id, "--nictype3", "Am79C973"]
        v.customize ["modifyvm", :id, "--nictype4", "Am79C973"]
        v.customize ["modifyvm", :id, "--nicpromisc1", "allow-all"]
        v.customize ["modifyvm", :id, "--nicpromisc2", "allow-all"]
        v.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
        v.customize ["modifyvm", :id, "--nicpromisc4", "allow-all"]
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]

      end

      config.vm.network "private_network",
      ip: node['mgmt_ip'],
      virtualbox__intnet: "LABS"


      config.vm.network "forwarded_port", guest: 443, host: 30443, auto_correct: true
      config.vm.network "forwarded_port", guest: 80, host: 30080, auto_correct: true
      

      config.vm.network "public_network",
      bridge: ["enp4s0","wlp3s0","enp3s0f1","wlp2s0"],
      auto_config: true

      config.vm.provision "shell", inline: <<-SHELL
      DEBIAN_FRONTEND=noninteractive apt-get update -qq && DEBIAN_FRONTEND=noninteractive apt-get install -qq chrony && timedatectl set-timezone Europe/Madrid
      SHELL

      config.vm.provision :shell, :inline => update_hosts

      config.vm.provision "shell", inline: <<-SHELL
        sudo cp -R /src ~ubuntu
        sudo chown -R ubuntu:ubuntu ~ubuntu/src
      SHELL
 


      if node['role'] == "client"
      	config.vm.provision "shell", inline: <<-SHELL
          apt-get install -qq curl 
          DEBIAN_FRONTEND=noninteractive apt-get install -qq lxde xinit firefox unzip zip gpm mlocate console-common chromium-browser
          service gpm start
          update-rc.d gpm enable
          localectl set-x11-keymap es
          localectl set-keymap es
          setxkbmap -layout es
          echo -e "XKBLAYOUT=\"es\"\nXKBMODEL=\"pc105\"\nXKBVARIANT=\"\"\nXKBOPTIONS=\"lv3:ralt_switch,terminate:ctrl_alt_bksp\"" >/etc/default/keyboard
          echo '@setxkbmap -option lv3:ralt_switch,terminate:ctrl_alt_bksp "es"' | sudo tee -a /etc/xdg/lxsession/LXDE/autostart
          echo '@setxkbmap -layout "es"'|tee -a /etc/xdg/lxsession/LXDE/autostart
        SHELL
	      next
      end




      ## INSTALL NGINX --> on script because we can reprovision
      config.vm.provision "shell", inline: <<-SHELL
        if [ -f /nginx_keys/nginx-repo.key ]
        then
          mkdir -p /etc/ssl/nginx
          cp -pR /nginx_keys/nginx-repo* /etc/ssl/nginx
          chown -R root:root /etc/ssl/nginx
        fi
        DEBIAN_FRONTEND=noninteractive apt-get install -qq \
        apt-transport-https \
        ca-certificates \
        lsb-release \
        curl \
        software-properties-common
        wget -q  -O /tmp/nginx_signing.key http://nginx.org/keys/nginx_signing.key
        apt-key add /tmp/nginx_signing.key && rm -rf /tmp/nginx_signing.key
        if [ -f /etc/ssl/nginx/nginx-repo.key ]
        then
          add-apt-repository "deb [arch=amd64] https://plus-pkgs.nginx.com/ubuntu $(lsb_release -cs) nginx-plus"
          wget -q -O /etc/apt/apt.conf.d/90nginx https://cs.nginx.com/static/files/90nginx
          DEBIAN_FRONTEND=noninteractive apt-get update -qq
          DEBIAN_FRONTEND=noninteractive apt-get install -qq nginx-plus
        else
          add-apt-repository "deb [arch=amd64] http://nginx.org/packages/mainline/ubuntu/ $(lsb_release -cs) nginx"
          DEBIAN_FRONTEND=noninteractive apt-get update -qq
          DEBIAN_FRONTEND=noninteractive apt-get install -qq nginx                 
        fi
        systemctl restart nginx
      SHELL

   end
  end

end
