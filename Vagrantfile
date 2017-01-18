# (c) 2016 DataNexus Inc.  All Rights Reserved
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'optparse'
require 'resolv'

options = {}
optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:kafka_addr] = nil
  opts.on( '-k', '--kafka-addr IP_ADDR', 'IP_ADDR of the kafka server' ) do |kafka_addr|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:kafka_addr] = kafka_addr.gsub(/^=/,'')
  end

  options[:influxdb_addr] = nil
  opts.on( '-i', '--influxdb-addr IP_ADDR', 'IP_ADDR of the influxdb server' ) do |influxdb_addr|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-i=192.168.1.1')
    options[:influxdb_addr] = influxdb_addr.gsub(/^=/,'')
  end

  options[:ip_addr] = nil
  opts.on( '-a', '--addr IP_ADDR', 'IP_ADDR of the provisioned server' ) do |ip_addr|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-i=192.168.1.1')
    options[:ip_addr] = ip_addr.gsub(/^=/,'')
  end

  opts.on_tail( '-h', '--help', 'Display this screen' ) do
    print opts
    exit
  end

end

begin
  optparse.parse!
rescue SystemExit
  ;
rescue Exception => e
  print "ERROR: could not parse command (#{e.message})\n"
  print optparse
  exit 1
end

if (!options[:ip_addr] || !options[:kafka_addr] || !options[:influxdb_addr]) && (ARGV[0] == "up" || ARGV[0] == "provision")
  print "ERROR; IP_ADDR of the provisioned server, influxdb server and kafka server must be supplied for 'up' and 'provision' commands\n"
  print optparse
  exit 1
elsif options[:kafka_addr] && !(options[:kafka_addr] =~ Resolv::IPv4::Regex)
  print "ERROR; input kafka server IP address '#{options[:kafka_addr]}' is not a valid IP address"
  exit 2
elsif options[:influxdb_addr] && !(options[:influxdb_addr] =~ Resolv::IPv4::Regex)
  print "ERROR; input influxdb server IP address '#{options[:influxdb_addr]}' is not a valid IP address"
  exit 2
elsif options[:ip_addr] && !(options[:ip_addr] =~ Resolv::IPv4::Regex)
  print "ERROR; input server IP address '#{options[:ip_addr]}' is not a valid IP address"
  exit 2
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
proxy = ENV['http_proxy'] || ""
no_proxy = ENV['no_proxy'] || ""
Vagrant.configure("2") do |config|
  if Vagrant.has_plugin?("vagrant-proxyconf")
    if defined?(HTTP_PROXY)
      config.proxy.http               = $proxy
      config.proxy.no_proxy           = "localhost,127.0.0.1"
      config.vm.box_download_insecure = true
      config.vm.box_check_update      = false
    end
    if $no_proxy
      config.proxy.no_proxy           = $no_proxy
    end
  end
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "centos/7"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Create a private network, which allows host-only access to the machine
  # using a specific IP.
  config.vm.network "private_network", ip: "#{options[:ip_addr]}"

  # Create a public network, which generally matched to bridged network.
  # Bridged networks make the machine appear as another physical device on
  # your network.
  # config.vm.network "public_network"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"
  config.vm.synced_folder ".", "/vagrant"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Define a Vagrant Push strategy for pushing to Atlas. Other push strategies
  # such as FTP and Heroku are also available. See the documentation at
  # https://docs.vagrantup.com/v2/push/atlas.html for more information.
  # config.push.define "atlas" do |push|
  #   push.app = "YOUR_ATLAS_USERNAME/YOUR_APPLICATION_NAME"
  # end

  config.vm.define "#{options[:ip_addr]}"
  telegraf_addr_array = "#{options[:ip_addr]}".split(/,\w*/)

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "site.yml"
    ansible.extra_vars = {
      proxy_env: { http_proxy: proxy, no_proxy: no_proxy},
      kafka_addr: "#{options[:kafka_addr]}",
      influxdb_addr: "#{options[:influxdb_addr]}",
      input_filter: 'cpu:disk:diskio:kernel:mem:processes:swap:system',
      output_filter: 'kafka:influxdb',
      host_inventory: telegraf_addr_array
    }
  end

end
