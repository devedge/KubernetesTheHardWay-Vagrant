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

### Load Balancer - haproxy
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

### Controllers
Add the three controller nodes, which are numbered 0-2.

```ruby
    ...

    (0..2).each do |n|
      config.vm.define "controller-#{n}" do |c|
          c.vm.hostname = "controller-#{n}"
          c.vm.network "private_network", ip: "192.168.199.1#{n}"
  
          # commented out, to do manually
          # c.vm.provision :shell, :path => "scripts/build/vagrant-setup-hosts-file.bash"
  
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

### Workers
Add three worker nodes to the Vagrantfile. These will also be numbered 0-2.

```ruby
    ...
  
    (0..2).each do |n|
      config.vm.define "worker-#{n}" do |c|
          c.vm.hostname = "worker-#{n}"
          c.vm.network "private_network", ip: "192.168.199.2#{n}"

          # c.vm.provision :shell, :path => "scripts/build/vagrant-setup-routes.bash"
          # c.vm.provision :shell, :path => "scripts/build/vagrant-setup-hosts-file.bash"
      end
    end

    ...
```

Set up the hosts file just as in the controllers. Then, run these commands in each respective node to configure networking routes:

**`worker-0`**
```
sudo route add -net 10.21.0.0/16 gw 192.168.199.21
sudo route add -net 10.22.0.0/16 gw 192.168.199.22
```

**`worker-1`**
```
sudo route add -net 10.20.0.0/16 gw 192.168.199.20
sudo route add -net 10.22.0.0/16 gw 192.168.199.22
```

**`worker-2`**
```
sudo route add -net 10.20.0.0/16 gw 192.168.199.20
sudo route add -net 10.21.0.0/16 gw 192.168.199.21
```

### Loadbalancer, Traefik
Set up the load balancer, which will be used later.

```ruby
    ...

    config.vm.define "traefik-0", autostart: false do |c|
        c.vm.hostname = "traefik-0"
        c.vm.network "private_network", ip: "192.168.199.30"
  
        # c.vm.provision :shell, :path => "scripts/build/vagrant-setup-routes.bash"
    end
```

In the load balancer, add routes to every worker node:
```
sudo route add -net 10.20.0.0/16 gw 192.168.199.20
sudo route add -net 10.21.0.0/16 gw 192.168.199.21
sudo route add -net 10.22.0.0/16 gw 192.168.199.22
```

## Configuring PKI Infrastructure
Then, we'll set up a PKI infrastructure for the cluster, using the `cfssl` tool. A certificate authority will be created, and then TLS will be generated for a number of Kubernetes components.

### Creating the Certificate Authority
Create 2 JSON files:

**`ca-config.json`**
```json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
```

**`ca-csr.json`**
```json
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
```

Then, generate a new CA Key and certificate with:

`$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca`

Three new files will be created in the current directory, `ca.csr`, `ca.pem`, and `ca-key.pem`

