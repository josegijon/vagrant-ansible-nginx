Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.network "private_network", ip: "192.168.57.57"

  config.vm.provision "ansible_local" do |ansible|
    ansible.playbook = "ansible/playbook.yml"
  end
end