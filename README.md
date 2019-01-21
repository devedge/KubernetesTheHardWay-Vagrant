# KubernetesTheHardWay-Vagrant
Following the "[Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)" tutorial, using Vagrant &amp; VirtualBox

This is my walkthrough of manually setting up [Kinvolk's version](https://github.com/kinvolk/kubernetes-the-hard-way-vagrant), which uses [`cri-o`](https://cri-o.io) instead of [`containerd`](https://github.com/containerd/containerd) for the runtime interface.

## Setup
Required tools:

`brew cask install virtualbox`

`brew cask install vagrant`

`brew install cfssl`

`brew install kubectl`


## Vagrantfile
The first step is to initialize a Vagrantfile. Here, the VM is ubuntu-16.04 (using the recommended `bento` image), and the default memory is set at 512MB. There is no officially supported way to specify initial disk size.

```ruby
Vagrant.configure("2") do |config|
  # Use bento's ubuntu machine instead
  config.vm.box = "bento/ubuntu-16.04"
  config.vm.box_version = "201812.27.0"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "512"
    vb.customize ["modifyvm", :id, "--audio", "none"]
  end

end
```

## Load Balancer - haproxy
Next, add the load balancer. The script for autocompletion is commented out now, so we can configure haproxy manually.

```ruby
    ...

    # Load balancer, haproxy
    config.vm.define "lb-0" do |c|
      c.vm.hostname = "lb-0"
      c.vm.network "private_network", ip: "192.168.199.40"

      # will be done manually first
      # c.vm.provision :shell, :path => "scripts/build/vagrant-setup-haproxy.bash"

      c.vm.provider "virtualbox" do |vb|
        vb.memory = "256"
      end
    end

    ...
```

Bring up the current Vagrant configuration, which is just the load balancer:

`$ vagrant up`

and ssh into the box with:

`$ vagrant ssh lb-0`

Update the package definitions, and install haproxy:

`sudo apt-get update`

`sudo apt-get install -y haproxy`


In the haproxy configuration file located at `/etc/haproxy/haproxy.cfg`, under the `defaults` section, ensure that `mode` is set to `tcp`, and `option` is set to `tcplog`. Then, add the next 2 sections:

```
frontend k8s
    bind 192.168.199.40:6443
    default_backend k8s_backend

backend k8s_backend
    balance roundrobin
    mode tcp
    server controller-0 192.168.199.10:6443 check inter 1000
    server controller-1 192.168.199.11:6443 check inter 1000
    server controller-2 192.168.199.12:6443 check inter 1000
```
_The configuration script can be found under `scripts/build/vagrant-setup-haproxy.bash`_

Restart haproxy with `sudo systemctl restart haproxy`

## Controllers
Add the three controller nodes, which are numbered 0-2.

```ruby
    ...

    (0..2).each do |n|
      config.vm.define "controller-#{n}" do |c|
          c.vm.hostname = "controller-#{n}"
          c.vm.network "private_network", ip: "192.168.199.1#{n}"
  
          # commented out, to do manually
          # c.vm.provision :shell, :path => "scripts/vagrant-setup-hosts-file.bash"
  
          c.vm.provider "virtualbox" do |vb|
            vb.memory = "640"
          end
      end
    end

    ...
```

`ssh` into each of the controller nodes, and append this to the `/etc/hosts` file:

```
# KTHW Vagrant machines
192.168.199.10 controller-0
192.168.199.11 controller-1
192.168.199.12 controller-2
192.168.199.20 worker-0
192.168.199.21 worker-1
192.168.199.22 worker-2
```

