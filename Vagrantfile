# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :server=> {
        :box_name => "centos/7",
        :ip_addr => '192.168.27.150'
  },
  :replica => {
        :box_name => "centos/7",
        :ip_addr => '192.168.27.151'
  },
  :backup => {
        :box_name => "centos/7",
        :ip_addr => '192.168.27.152'
  }
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
            vb.customize ["modifyvm", :id, "--memory", "1024"]
            # Подключаем дополнительные диски
            #vb.customize ['createhd', '--filename', second_disk, '--format', 'VDI', '--size', 5 * 1024]
            #vb.customize ['storageattach', :id, '--storagectl', 'IDE', '--port', 0, '--device', 1, '--type', 'hdd', '--medium', second_disk]
          end
          case boxname.to_s
          when "backup"
              box.vm.provision "ansible" do |ansible|
                  ansible.compatibility_mode = "2.0"
                  ansible.playbook = "pg.yml"
                  ansible.verbose = "true"
                  ansible.become = "true"
                  ansible.limit = "all"
              end
          end
      end
  end
end
