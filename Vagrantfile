Vagrant.configure("2") do |config|
    # This will be applied to every vagrant file that comes after it
    config.vm.box = "bento/ubuntu-20.04"
    ## Fleet
    config.vm.define "fleet" do |fleet_server|
      fleet_server.vm.provision "shell", path: "script.sh"
      fleet_server.vm.network "private_network", ip: "172.16.128.21" 
      fleet_server.vm.hostname = "fleet"
      fleet_server.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--audio", "none"]
        v.memory = 4024
        v.cpus = 2
      end
    end
    ## Fleet Worker Node 1
    config.vm.define "fleet_worker1" do |fleet_worker|
        fleet_worker.vm.provision "shell", path: "script.sh"
        fleet_worker.vm.network "private_network", ip: "172.16.128.22"
        fleet_worker.vm.hostname = "fleetworker1"
        fleet_worker.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--audio", "none"]
        v.memory = 4024
        v.cpus = 2
        end
    end
    ## Downstream Single Node Cluster 1
    config.vm.define "downstream1_master" do |downstream1_master|
      downstream1_master.vm.provision "shell", path: "script.sh"
      downstream1_master.vm.network "private_network", ip: "192.168.10.5" 
      downstream1_master.vm.hostname = "downstream1master"
      downstream1_master.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--audio", "none"]
        v.memory = 4024
        v.cpus = 2
      end
    end
    ## Downstream Single Node Cluster 1
    config.vm.define "downstream1_worker" do |downstream1_worker|
      downstream1_worker.vm.provision "shell", path: "script.sh"
      downstream1_worker.vm.network "private_network", ip: "192.168.10.6" 
      downstream1_worker.vm.hostname = "downstream1worker"
      downstream1_worker.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--audio", "none"]
        v.memory = 4024
        v.cpus = 2
      end
    end
end