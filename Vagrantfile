Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"

  # VM 1 - master1
  config.vm.define "master1" do |m|
    m.vm.hostname = "master1"
    m.vm.network "private_network", ip: "192.168.56.10"
    m.vm.provider "virtualbox" do |vb|
      vb.memory = 512
      vb.cpus = 1
    end
  end

  # VM 2 - master2
  config.vm.define "master2" do |m|
    m.vm.hostname = "master2"
    m.vm.network "private_network", ip: "192.168.56.11"
    m.vm.provider "virtualbox" do |vb|
      vb.memory = 512
      vb.cpus = 1
    end
  end
end