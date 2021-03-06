Vagrant.configure("2") do |config|
  config.vm.box = "precise64"

  # Ubuntu 12.04.3 LTS base with 3.8 kernel (ready for Docker)
  config.vm.box_url = "https://oss-binaries.phusionpassenger.com/vagrant/boxes/ubuntu-12.04.3-amd64-vbox.box"
  
  # Avahi-daemon will broadcast the server's address as deis-controller.local
  config.vm.host_name = "deis-controller"

  # IP will be associated to 'deis-controller.local' using avahi-daemon
  config.vm.network :private_network, ip: "192.168.61.100"
  
  # The Deis Controller requires at least 2G of RAM to install.
  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", "2048"]
  end

  # 'deis provider:discover' detects the host machine's user and IP address, however, that command cannot
  # be guareteed to run inside the deis codebase. Therefore we can't use that opportunity to discover
  # the path of the codebase on the host machine. Therefore we do it now as this Vagrantfile has to exist
  # inside the codebase.
  nodes_dir = File.dirname(__FILE__) + '/contrib/vagrant/nodes'

  config.vm.provision :shell, inline: <<-SCRIPT
    # install latest stable chef for subsequent provision blocks
    sudo apt-get install -yq curl
    chef-client -v | grep 10.14.2 && curl -L https://www.opscode.com/chef/install.sh | sudo bash
    # install 'etcd' gem using the vagrant chef runtime
    sudo /opt/chef/embedded/bin/gem install etcd --no-ri --no-rdoc
    # Avahi-daemon broadcasts the machine's hostname to local DNS.
    # Therefore 'deis-controller.local' in this case.
    sudo apt-get install -yq avahi-daemon
    # Make a record of where the deis code base is on the host machine
    echo "#{nodes_dir}" > /home/vagrant/.host_nodes_dir
  SCRIPT
  
  # load chef config from ~/.chef/knife.rb (requires `vagrant plugin install chef`)
  Chef::Config.from_file(File.join(ENV['HOME'], '.chef', 'knife.rb'))
  
  config.vm.provision "chef_client" do |chef|
    chef.chef_server_url = Chef::Config[:chef_server_url]
    chef.client_key_path = Chef::Config[:client_key]
    chef.validation_client_name = Chef::Config[:validation_client_name]
    chef.validation_key_path = Chef::Config[:validation_key]
    chef.log_level = Chef::Config[:log_level]
    # TODO: replace with in-recipe public_ip lookup that handles vagrant/ec2/metal
    chef.json = {
        "deis" => {
            "public_ip" => "192.168.61.100",
            "dev" => {
              "mode" => true,
              "source" => "/vagrant"
            }
        }
    }
    # define the run list
    chef.add_recipe 'deis::controller'
  end

end

# If you want to do some funky custom stuff to your box, but don't want those things tracked by git,
# add a Vagrantfile.local and it will be included. You can use the exact same syntax as above. For
# example you could mount your dev version of deis onto the VM and hack live on the VM;
# `config.vm.share_folder "deis", "/opt/deis", "~/myworkspace/deis"
# Or if you're low on RAM you can boot the VM with less RAM. Note that at least 1GB is needed for
# installation, but you may be able to get away with 512MB once everything is installed.
load "Vagrantfile.local" if File.exists? "Vagrantfile.local"
