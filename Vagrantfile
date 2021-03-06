# -*- mode: ruby -*-
# vi: set ft=ruby :
#

CLUSTERFILE = 'Clusterfile.yaml'

if not File.exists? CLUSTERFILE
  STDERR.puts "#{CLUSTERFILE} not found"
  exit 1
end

$cluster = YAML.load_file(CLUSTERFILE)


Vagrant::Config.run do |config|

  vm_default = proc do |cfg|
    cfg.vm.box = "precise64-ruby-1.9.3-p194"
    cfg.vm.box_url = "https://dl.dropbox.com/u/14292474/vagrantboxes/precise64-ruby-1.9.3-p194.box"

    cfg.vm.customize ["modifyvm", :id, "--rtcuseutc", "on"]
    cfg.vm.customize ["modifyvm", :id, "--memory", 256]

    # https://groups.google.com/forum/?fromgroups#!topic/vagrant-up/a2COzF4E0gc%5B1-25%5D
    # http://serverfault.com/questions/414517/vagrant-virtualbox-host-only-multiple-node-networking-issue
    # cfg.vm.customize ["modifyvm", :id, "--nicpromisc3", "allow-all"]
  end


  chef_default = proc do |chef|
    # local, solo, opscode's
    chef.cookbooks_path = ["cookbooks", "../cookbooks.solo", "../cookbooks/chef-server"]
    chef.roles_path     = "roles"
    chef.data_bags_path = "data_bags"
  end


  # configure nodes
  $cluster[:nodes].each do |node,opts|
    puts "Load configuration for #{node}"
    config.vm.define node.to_s do |cfg|
      # defaults
      vm_default[cfg]

      cfg.vm.customize ["modifyvm", :id, "--cpus", opts[:cpus]]

      cfg.vm.host_name = node.to_s

      if not opts[:hostonly].nil? then
        cfg.vm.network :hostonly, opts[:hostonly]
      end
      if not opts[:bridged].nil? then
        cfg.vm.network :bridged, opts[:bridged]
      end

      # forwarding
      if not opts[:forward].nil? then
        opts[:forward].each { |p| cfg.vm.forward_port p[0], p[1]}
      end

      # vb guest version check
      if defined? cfg.vbguest
        cfg.vbguest.auto_update = false
        cfg.vbguest.no_remote   = true
      end

      # only for hosted chef
      if not opts[:chef_client].nil? then
        puts "Chef client: #{node}"
        orgname = opts[:chef_client][:orgname]

        cfg.vm.provision :chef_client do |chef|
          chef.chef_server_url = "https://api.opscode.com/organizations/#{orgname}"
          chef.validation_key_path = ".chef/#{orgname}-validator.pem"
          chef.validation_client_name = "#{orgname}-validator"
          chef.encrypted_data_bag_secret_key_path = ".chef/data_bag.key"
          chef.node_name = "#{node.to_s}"
          chef.log_level = :debug
          chef.environment = opts[:chef_client][:environment]

          if not opts[:chef_client][:roles].nil? then
            chef.run_list = opts[:chef_client][:roles].split(",")
          end

        end # :chef_client
      else
        cfg.vm.provision :chef_solo do |chef|
          puts "Chef solo: #{node}"
          # chef_default[chef]
          chef.cookbooks_path = opts[:cookbooks_path].nil? ? "cookbooks" : opts[:cookbooks_path]

          # chef.environment = "_default"
          chef.log_level = :debug

          # process role string
          if not opts[:chef_solo][:roles].nil? then 
            chef.run_list = opts[:chef_solo][:roles].split(",")
          end

          # access from chef node[:]
          # todo
          chef.json = {
            :cluster => $cluster,
            :host => opts,
            "chef_server" => opts[:chef_server]
          }
        end
      end # :chef_solo

    end # cfg
  end # nodes

end
