Vagrant.configure("2") do |config|
  # Use bento's ubuntu machine instead
  config.vm.box = "bento/ubuntu-16.04"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "512"
    vb.customize ["modifyvm", :id, "--audio", "none"]
  end

  # Load balancer, haproxy
  config.vm.define "lb-0" do |c|
      c.vm.hostname = "lb-0"
      c.vm.network "private_network", ip: "192.168.199.40"

      # install and set up haproxy
      c.vm.provision :shell, :path => "scripts/build/vagrant-setup-haproxy.bash"

      c.vm.provider "virtualbox" do |vb|
        vb.memory = "256"
      end
  end

end
