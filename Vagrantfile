# (c) 2016 DataNexus Inc.  All Rights Reserved
# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'optparse'
require 'resolv'

# monkey-patch that is used to leave unrecognized options in the ARGV
# list so that they can be processed by underlying vagrant command
class OptionParser
  # Like order!, but leave any unrecognized --switches alone
  def order_recognized!(args)
    extra_opts = []
    begin
      order!(args) { |a| extra_opts << a }
    rescue OptionParser::InvalidOption => e
      extra_opts << e.args[0]
      retry
    end
    args[0, 0] = extra_opts
  end
end

options = {}
# vagrant commands that include these commands can be run without specifying
# any IP addresses
no_ip_commands = ['version', 'global-status', '--help', '-h']
# vagrant commands that only work for a single IP address
single_ip_commands = ['status', 'ssh']
# vagrant command arguments that indicate whether or not we are provisioning nodes
provisioning_command_args = ['up', 'provision']
no_kafka_required_command_args = ['destroy']
not_provisioning_flag = ['--no-provision']

optparse = OptionParser.new do |opts|
  opts.banner    = "Usage: #{opts.program_name} [options]"
  opts.separator "Options"

  options[:addr_list] = nil
  opts.on( '-a', '--addr-list A1,A2[,...]', 'Telegraf address list' ) do |addr_list|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-a=192.168.1.1')
    options[:addr_list] = addr_list.gsub(/^=/,'')
  end

  options[:kafka_list] = nil
  opts.on( '-k', '--kafka-list A1,A2[,...]', 'Kafka address list' ) do |kafka_list|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=192.168.1.1')
    options[:kafka_list] = kafka_list.gsub(/^=/,'')
  end

  options[:kafka_inventory_file] = nil
  opts.on( '-i', '--kafka-inventory-file FILE', 'Kafka (Ansible) inventory file' ) do |kafka_inventory_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-k=/tmp/vagrant_ansible_inventory')
    options[:kafka_inventory_file] = kafka_inventory_file.gsub(/^=/,'')
  end

  options[:telegraf_url] = nil
  opts.on( '-u', '--url TELEGRAF_URL', 'URL of a patched telegraf executable' ) do |telegraf_url|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-u=http://localhost/tmp.tgz')
    options[:telegraf_url] = telegraf_url.gsub(/^=/,'')
  end

  options[:local_telegraf_file] = nil
  opts.on( '-l', '--local-telegraf-file FILE', 'Path to a patched telegraf executable' ) do |local_telegraf_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-l=/tmp/telegraf')
    options[:local_telegraf_file] = local_telegraf_file.gsub(/^=/,'')
  end

  options[:local_vars_file] = nil
  opts.on( '-f', '--local-vars-file FILE', 'Local variables file' ) do |local_vars_file|
    # while parsing, trim an '=' prefix character off the front of the string if it exists
    # (would occur if the value was passed using an option flag like '-f=/tmp/local-vars-file.yml')
    options[:local_vars_file] = local_vars_file.gsub(/^=/,'')
  end

  opts.on_tail( '-h', '--help', 'Display this screen' ) do
    print opts
    exit
  end

end

begin
  optparse.order_recognized!(ARGV)
rescue SystemExit
  exit
rescue Exception => e
  print "ERROR: could not parse command (#{e.message})\n"
  print optparse
  exit 1
end

# check remaining arguments to see if the command requires
# an IP address (or not)
ip_required = (ARGV & no_ip_commands).empty?
# check the remaining arguments to see if we're provisioning or not
provisioning_command = !((ARGV & provisioning_command_args).empty?) && (ARGV & not_provisioning_flag).empty?
# and to see if multiple IP addresses are supported (or not) for the
# command being invoked
single_ip_command = !((ARGV & single_ip_commands).empty?)
# and to see if a kafka node (or cluster) must also be provided
no_kafka_required_command = !(ARGV & no_kafka_required_command_args).empty?


if options[:telegraf_url] && options[:local_telegraf_file]
  print "ERROR; the telegraf_url option and the local_telegraf_file options cannot be combined\n"
  exit 2
end

if options[:telegraf_url] && !(options[:telegraf_url] =~ URI::regexp)
  print "ERROR; input kafka URL '#{options[:telegraf_url]}' is not a valid URL\n"
  exit 3
end

if options[:local_telegraf_file] && !File.file?(options[:local_telegraf_file])
  print "ERROR; input local telegraf file path '#{options[:local_telegraf_file]}' is not a local file\n"
  exit 3
end

# if a kafka inventory file was included, then make sure the file exists
if options[:kafka_list] && !options[:kafka_inventory_file]
  print "ERROR; the if a kafka list is defined, a kafka inventory file must also be provided\n"
  exit 2
elsif options[:kafka_list] && !File.file?(options[:kafka_inventory_file])
  print "ERROR; input kafka inventory file '#{options[:kafka_inventory_file]}' is not a local file\n"
  exit 3
end

# if we're provisioning, then the `--addr-list` flag must be provided
telegraf_addr_array = []
kafka_addr_array = []
if provisioning_command || ip_required
  if !options[:addr_list]
    print "ERROR; IP address must be supplied (using the `-a, --addr-list` flag) for this vagrant command\n"
    exit 1
  else
    telegraf_addr_array = options[:addr_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
    if telegraf_addr_array.size == 1
      if !(telegraf_addr_array[0] =~ Resolv::IPv4::Regex)
        print "ERROR; input Telegraf IP address #{telegraf_addr_array[0]} is not a valid IP address\n"
        exit 2
      end
    elsif !single_ip_command
      # check the input `telegraf_addr_array` to ensure that all of the values passed in are
      # legal IP addresses
      not_ip_addr_list = telegraf_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
      if not_ip_addr_list.size > 0
        # if some of the values are not valid IP addresses, print an error and exit
        if not_ip_addr_list.size == 1
          print "ERROR; input Telegraf IP address #{not_ip_addr_list} is not a valid IP address\n"
          exit 2
        else
          print "ERROR; input Telegraf IP addresses #{not_ip_addr_list} are not valid IP addresses\n"
          exit 2
        end
      end
      # if this command requires a kafka node (or cluster), then parse the kafka_list and check it
      if !no_kafka_required_command
        if !options[:kafka_list]
          print "ERROR; IP address must be supplied (using the `-k, --kafka-list` flag) for this vagrant command\n"
          exit 1
        else
          kafka_addr_array = options[:kafka_list].split(',').map { |elem| elem.strip }.reject { |elem| elem.empty? }
          if kafka_addr_array.size == 1
            if !(kafka_addr_array[0] =~ Resolv::IPv4::Regex)
              print "ERROR; input Kafka IP address #{kafka_addr_array[0]} is not a valid IP address\n"
              exit 2
            end
          else
            # check the input `kafka_addr_array` to ensure that all of the values passed in are
            # legal IP addresses
            not_ip_addr_list = kafka_addr_array.select { |addr| !(addr =~ Resolv::IPv4::Regex) }
            if not_ip_addr_list.size > 0
              # if some of the values are not valid IP addresses, print an error and exit
              if not_ip_addr_list.size == 1
                print "ERROR; input kafka IP address #{not_ip_addr_list} is not a valid IP address\n"
                exit 2
              else
                print "ERROR; input kafka IP addresses #{not_ip_addr_list} are not valid IP addresses\n"
                exit 2
              end
            end
          end
        end
      end
    end
  end
end

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
if telegraf_addr_array.size > 0
  Vagrant.configure("2") do |config|
    proxy = ENV['http_proxy'] || ""
    no_proxy = ENV['no_proxy'] || ""
    proxy_username = ENV['proxy_username'] || ""
    proxy_password = ENV['proxy_password'] || ""
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
      if $proxy_username
        config.proxy.proxy_username     = $proxy_username
      end
      if $proxy_password
        config.proxy.proxy_password     = $proxy_password
      end
    end
    
    # Every Vagrant development environment requires a box. You can search for
    # boxes at https://atlas.hashicorp.com/search.
    config.vm.box = "centos/7"
    config.vm.box_check_update = false

    # loop through all of the addresses in the `telegraf_addr_array` and, if we're
    # creating VMs, create a VM for each machine; if we're just provisioning the
    # VMs using an ansible playbook, then wait until the last VM in the loop and
    # trigger the playbook runs for all of the nodes simultaneously using the
    # `site.yml` playbook
    telegraf_addr_array.each do |machine_addr|
      config.vm.define machine_addr do |machine|
        # create a private network, which each allow host-only access to the machine
        # using a specific IP.
        machine.vm.network "private_network", ip: machine_addr
        # if it's the last node in the list if input addresses, then provision
        # all of the nodes simultaneously (if the `--no-provision` flag was not
        # set, of course)
        if machine_addr == telegraf_addr_array[-1]
          # and provision our Telegraf node using the Ansible playbook in the site.yml file
          machine.vm.provision "ansible" do |ansible|
            ansible.limit = "all"
            ansible.playbook = "site.yml"
            ansible.groups = {
              telegraf: telegraf_addr_array
            }
            ansible.extra_vars = {
              proxy_env: {
                http_proxy: proxy,
                no_proxy: no_proxy,
                proxy_username: proxy_username,
                proxy_password: proxy_password
              },
              data_iface: "eth1",
              host_inventory: telegraf_addr_array,
              cloud: "vagrant"
            }
            # if defined, set the 'extra_vars[:telegraf_url]' value to the value that was passed in on
            # the command-line (eg. "https://10.0.2.2/tmp/telegraf")
            if options[:telegraf_url]
              ansible.extra_vars[:telegraf_url] = options[:telegraf_url]
            end
            # if defined, set the 'extra_vars[:local_telegraf_file]' value to the value that was passed
            # in on the command-line (eg. "/tmp/telegraf")
            if options[:local_telegraf_file]
              ansible.extra_vars[:local_telegraf_file] = options[:local_telegraf_file]
            end
            # if defined, set the 'extra_vars[:local_vars_file]' value to the value that was passed
            # in on the command-line (eg. "/tmp/local-vars.yml")
            if options[:local_vars_file]
              ansible.extra_vars[:local_vars_file] = options[:local_vars_file]
            end
            # if a kafka address list was passed in, then construct the inventory
            # hash we'll need in our playbook
            if kafka_addr_array.size > 0
              ansible.extra_vars[:kafka_nodes] = kafka_addr_array
              # use the contents of the kafka_inventory_file to construct the
              # inventory hash
              kafka_inventory = {}
              kafka_inventory_file = options[:kafka_inventory_file]
              File.open(kafka_inventory_file, "r") do |file|
                while (line = file.gets)
                  split_line = line.split
                  if split_line.size > 1 && kafka_addr_array.include?(split_line[0])
                    hostname = split_line[0]
                    inventory_hash = {}
                    for val in split_line[1..-1]
                      key,val = val.split('=')
                      if val
                        inventory_hash[key.to_sym] = val.delete("'")
                      end
                    end
                    kafka_inventory[hostname] = inventory_hash
                  end
                end
                ansible.extra_vars[:kafka_inventory] =  kafka_inventory
              end
            end
          end
        end
      end
    end
  end
end
