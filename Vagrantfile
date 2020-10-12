Vagrant.require_version ">= 1.7.0"

$os_image = (ENV['OS_IMAGE'] || "centos7").to_sym

def set_vbox(vb, config)
  vb.gui = false
  vb.memory = 2048
  vb.cpus = 2

  case $os_image
  when :centos7
    config.vm.box = "bento/centos-7.8"
  when :ubuntu16
    config.vm.box = "bento/ubuntu-16.04"
  end
end

Vagrant.configure("2") do |config|
  config.vm.provider "virtualbox"
  master = 1
  node = 4

  private_count = 10
  (1..(master + node)).each do |mid|
    name = (mid <= node) ? "n" : "m"
    id   = (mid <= node) ? mid : (mid - node)

    config.vm.define "k8s-#{name}#{id}" do |n|
      n.vm.hostname = "k8s-#{name}#{id}"
      ip_addr = "192.16.35.#{private_count}"
      n.vm.network :private_network, ip: "#{ip_addr}",  auto_config: true

      n.vm.provider :virtualbox do |vb, override|
        vb.name = "#{n.vm.hostname}"
        set_vbox(vb, override)
      end
      private_count += 1
    end
  end
  if Vagrant.has_plugin?("vagrant-cachier")
    config.cache.scope = :box
  end
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
    config.vbguest.no_install = true 
    config.vbguest.no_remote = true
  end
  # Install of dependency packages using script
  # config.vm.provision :shell, path: "./hack/setup-vms.sh"
end
