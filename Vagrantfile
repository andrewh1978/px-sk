# Edit these parameters
clusters = 2
nodes = 3
disk_size = 20
cluster_name = "px-test-cluster"
version = "2.3.1"
journal = false

# Set K8s version
k8s_version="1.16.3"

# Set AWS tags
tags = { :example_name => "example value" }

# Set some cloud-specific parameters
AWS_keypair_name = "ah"
AWS_sshkey_path = "#{ENV['HOME']}/.ssh/id_rsa"
AWS_type = "t3.large"
AWS_hostname_prefix = ""

# Do not edit below this line
if !File.exist?("id_rsa")
  system("ssh-keygen -t rsa -b 2048 -f id_rsa -N ''");
  File.delete("id_rsa.pub") if File.exist?("id_rsa.pub")
end

Vagrant.configure("2") do |config|

  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.provision "file", source: "id_rsa", destination: "/tmp/id_rsa"

  config.vm.box = "dummy"
  config.vm.provider :aws do |aws, override|
    aws.security_groups = ENV['AWS_sg']
    aws.keypair_name = AWS_keypair_name
    aws.region = ENV['AWS_region']
    aws.instance_type = AWS_type
    aws.ami = ENV['AWS_ami']
    aws.subnet_id = ENV['AWS_subnet']
    aws.associate_public_ip = true
    override.ssh.username = "centos"
    override.ssh.private_key_path = AWS_sshkey_path
  end

  env_ = { :cluster_name => cluster_name, :version => version, :journal => journal, :nodes => nodes, :clusters => clusters, :k8s_version => k8s_version }

  config.vm.provision "shell", path: "all-common", env: env_

  (1..clusters).each do |c|
    if c == 1
      platform = "swarm"
    else
      platform = "k8s"
    end
    if platform == "k8s"
      config.vm.provision "shell", path: "k8s-common"
    end

    hostname_master = "master-#{c}"
    ip_master = "192.168.#{100+c}.90"
    config.vm.hostname = hostname_master
    env = env_.merge({ :c => c, :ip_master => ip_master, :hostname_master => hostname_master })

    config.vm.define hostname_master do |master|

      master.vm.provider :aws do |aws|
        aws.private_ip_address = ip_master
        aws.tags = tags.merge({ :Name => "#{AWS_hostname_prefix}#{hostname_master}" })
        aws.block_device_mapping = [{ :DeviceName => "/dev/sda1", "Ebs.DeleteOnTermination" => true, "Ebs.VolumeSize" => 15 }]
      end
      master.vm.provision "shell", path: "#{platform}-master", env: env
    end

    (1..nodes).each do |n|
      config.vm.define "node-#{c}-#{n}" do |node|
        node.vm.hostname = "node-#{c}-#{n}"
        node.vm.provider :aws do |aws|
          aws.private_ip_address = "192.168.#{100+c}.#{100+n}"
          aws.tags = tags.merge({ :Name => "#{AWS_hostname_prefix}node-#{c}-#{n}" })
          aws.block_device_mapping = [{ :DeviceName => "/dev/sda1", "Ebs.DeleteOnTermination" => true, "Ebs.VolumeSize" => 15 }, { :DeviceName => "/dev/sdb", "Ebs.DeleteOnTermination" => true, "Ebs.VolumeSize" => disk_size }]
          if journal
            aws.block_device_mapping.push({ :DeviceName => "/dev/sdc", "Ebs.DeleteOnTermination" => true, "Ebs.VolumeSize" => 3 })
          end
        end
        node.vm.provision "shell", path: "#{platform}-node", env: env
      end
    end
  end
end
