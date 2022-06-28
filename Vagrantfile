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

            config.vm.provision "ansible" do |ansible|
            ansible.playbook = "playbook.yml"
            end            
        end
    end
end
    