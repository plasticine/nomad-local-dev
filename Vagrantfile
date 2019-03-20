# -*- mode: ruby -*-
# vi: set ft=ruby :

$NOMAD_VERSION = "0.9.0-beta3"
$CONSUL_VERSION = "1.4.2"
$FIRECRACKER_VERSION = "0.15.0"

$base = <<BASE
# Update apt and get dependencies
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y unzip curl vim jq netcat \
    apt-transport-https \
    ca-certificates \
    software-properties-common \
    qemu-kvm \
    libvirt-clients \
    libvirt-daemon-system \
    bridge-utils \
    libguestfs-tools \
    genisoimage \
    virtinst \
    libosinfo-bin
BASE

$docker_install = <<DOCKER_INSTALL
echo "Installing Docker..."
if [[ -f /etc/apt/sources.list.d/docker.list ]]; then
    echo "Docker repository already installed; Skipping"
else
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    sudo apt-get update
fi
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y docker-ce
# Restart docker to make sure we get the latest version of the daemon if there is an upgrade
sudo service docker restart
# Make sure we can actually use docker as the vagrant user
sudo usermod -aG docker vagrant
DOCKER_INSTALL

$dnsmasq = <<DNSMASQ
YUM=$(which yum 2>/dev/null)
APT_GET=$(which apt-get 2>/dev/null)
if [[ ! -z ${YUM} ]]; then
  logger "Installing dnsmasq"
  sudo yum install -q -y dnsmasq
elif [[ ! -z ${APT_GET} ]]; then
  logger "Installing dnsmasq"
  sudo apt-get -qq -y update
  sudo apt-get install -qq -y dnsmasq-base dnsmasq
else
  logger "Dnsmasq not installed due to OS detection failure"
  exit 1;
fi

logger "Configuring dnsmasq to forward .consul requests to consul port 8600"
sudo sh -c 'echo "server=/consul/127.0.0.1#8600" >> /etc/dnsmasq.d/consul'
sudo systemctl enable dnsmasq
sudo systemctl restart dnsmasq
DNSMASQ

$firecracker_download = <<FIRECRACKER_DOWNLOAD
FIRECRACKER_VERSION=#{$FIRECRACKER_VERSION}

echo "Fetching Firecracker..."
if [[ ! -f /tmp/firecracker ]] ; then
    cd /tmp/
    sudo curl -fL https://github.com/firecracker-microvm/firecracker/releases/download/v${FIRECRACKER_VERSION}/firecracker-v${FIRECRACKER_VERSION} -o firecracker
    sudo curl -fL https://github.com/firecracker-microvm/firecracker/releases/download/v${FIRECRACKER_VERSION}/jailer-v${FIRECRACKER_VERSION} -o jailer
fi
FIRECRACKER_DOWNLOAD

$nomad_download = <<NOMAD_DOWNLOAD
NOMAD_VERSION=#{$NOMAD_VERSION}

echo "Fetching Nomad..."
if [[ ! -f /tmp/nomad ]] ; then
    cd /tmp/
    sudo curl https://releases.hashicorp.com/nomad/${NOMAD_VERSION}/nomad_${NOMAD_VERSION}_linux_amd64.zip -o nomad.zip
    sudo unzip nomad.zip
fi
NOMAD_DOWNLOAD

$consul_download = <<CONSUL_DOWNLOAD
CONSUL_VERSION=#{$CONSUL_VERSION}

 echo "Fetching Consul..."
if [[ ! -f /tmp/consul ]] ; then
    cd /tmp/
    sudo curl https://releases.hashicorp.com/consul/${CONSUL_VERSION}/consul_${CONSUL_VERSION}_linux_amd64.zip -o consul.zip
    sudo unzip consul.zip
fi
CONSUL_DOWNLOAD

$firecracker_install = <<FIRECRACKER_INSTALL
echo "Installing Firecracker..."
sudo install /tmp/firecracker /usr/bin/firecracker
sudo install /tmp/jailer /usr/bin/jailer

sudo curl -fsSL -o /root/hello-vmlinux.bin https://s3.amazonaws.com/spec.ccfc.min/img/hello/kernel/hello-vmlinux.bin
sudo curl -fsSL -o /root/hello-rootfs.ext4 https://s3.amazonaws.com/spec.ccfc.min/img/hello/fsfiles/hello-rootfs.ext4

sudo rm -f /tmp/firecracker
sudo rm -f /tmp/jailer
FIRECRACKER_INSTALL

$nomad_install = <<NOMAD_INSTALL
echo "Installing Nomad..."
sudo install /tmp/nomad /usr/bin/nomad
(
cat <<-EOF
[Unit]
Description=nomad agent
Requires=network-online.target
After=network-online.target

[Service]
Restart=on-failure
ExecStart=/usr/bin/nomad agent -config /etc/nomad.d/
ExecReload=/bin/kill -HUP $MAINPID

[Install]
WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/nomad.service

cd /tmp
sudo mkdir -p /etc/nomad.d
sudo chmod a+w /etc/nomad.d

# Set hostname's IP to made advertisement Just Work
sudo sed -i -e "s/.*nomad.*/$(ip route get 1 | awk '{print $NF;exit}') nomad/" /etc/hosts

nomad -autocomplete-install
sudo rm -f /tmp/nomad.zip
NOMAD_INSTALL


$consul_install = <<CONSUL_INSTALL
echo "Installing Consul..."
sudo install /tmp/consul /usr/bin/consul
(
cat <<-EOF
  [Unit]
  Description=consul agent
  Requires=network-online.target
  After=network-online.target

  [Service]
  Restart=on-failure
  ExecStart=/usr/bin/consul agent -config-dir=/etc/consul.d
  ExecReload=/bin/kill -HUP $MAINPID

  [Install]
  WantedBy=multi-user.target
EOF
) | sudo tee /etc/systemd/system/consul.service

sudo mkdir -p /etc/consul.d
sudo chmod a+w /etc/consul.d
sudo rm -f /tmp/consul.zip
CONSUL_INSTALL

$nomad_config = <<NOMAD_CONFIG
INSTANCE_IP="$(/sbin/ifconfig eth1 | grep 'inet addr:' | awk '{print substr($2,6)}')"

sudo cat << EOF > /etc/nomad.d/config.hcl
data_dir = "/opt/nomad/data"
advertise {
  http = "$INSTANCE_IP"
  rpc  = "$INSTANCE_IP"
  serf = "$INSTANCE_IP"
}
log_level = "DEBUG"
server {
  enabled = true
  bootstrap_expect = 1
}
client {
  enabled = true
  options {
    "docker.cleanup.image"   = "0"
    "driver.raw_exec.enable" = "1"
  }
}
plugin "nomad-plugin-firecracker-linux-amd64" {
  config {
    firecracker_path = "/usr/bin/firecracker"
    jailer_path = "/usr/bin/jailer"
    use_jailer = false
  }
}
consul {
  address = "127.0.0.1:8500"
}
EOF
NOMAD_CONFIG

$consul_config = <<CONSUL_CONFIG
sudo cat << EOF > /etc/consul.d/config.json
{
  "server": true,
  "bootstrap_expect": 1,
  "leave_on_terminate": true,
  "advertise_addr": "$(/sbin/ifconfig eth1 | grep 'inet addr:' | awk '{print substr($2,6)}')",
  "retry_join": ["192.168.60.150","192.168.60.151","192.168.60.152"],
  "data_dir": "/opt/consul/data",
  "client_addr": "0.0.0.0",
  "log_level": "INFO",
  "ui": true
}
EOF
CONSUL_CONFIG

$consul_online_target = <<CONSUL_ONLINE_TARGET
sudo cat << EOF > /etc/systemd/system/consul-online.target
[Unit]
Description=Consul Online
RefuseManualStart=true
EOF
CONSUL_ONLINE_TARGET

$consul_online_service = <<CONSUL_ONLINE_SERVICE
sudo cat << EOF > /etc/systemd/system/consul-online.service
[Unit]
Description=Consul Online
Requires=consul.service
After=consul.service
[Service]
Type=oneshot
ExecStart=/usr/bin/consul-online.sh
User=consul
Group=consul
[Install]
WantedBy=consul-online.target multi-user.target
EOF
CONSUL_ONLINE_SERVICE

$consul_online_script = <<CONSUL_ONLINE_SCRIPT
sudo cat << EOF > /usr/bin/consul-online.sh
#!/usr/bin/env bash
set -e
set -o pipefail
CONSUL_ADDRESS=${1:-"127.0.0.1:8500"}

function waitForConsulToBeAvailable() {
  local consul_addr=$1
  local consul_leader_http_code
  consul_leader_http_code=$(curl --silent --output /dev/null --write-out "%{http_code}" "${consul_addr}/v1/operator/raft/configuration") || consul_leader_http_code=""
  while [ "x${consul_leader_http_code}" != "x200" ] ; do
    echo "Waiting for Consul to get a leader..."
    sleep 5
    consul_leader_http_code=$(curl --silent --output /dev/null --write-out "%{http_code}" "${consul_addr}/v1/operator/raft/configuration") || consul_leader_http_code=""
  done
}
waitForConsulToBeAvailable "${CONSUL_ADDRESS}"
EOF
CONSUL_ONLINE_SCRIPT

Vagrant.configure(2) do |config|
  config.vm.provider :vmware_desktop do |vmware|
    vmware.gui = false
    vmware.vmx["memsize"] = "2048"
    vmware.vmx["numvcpus"] = "2"
    vmware.vmx["vhv.enable"] = "TRUE"
  end

  config.vm.define "node" do |node|
    node.vm.network :private_network, ip: "192.168.60.150"
    node.vm.box = "bento/ubuntu-18.04"
    node.vm.hostname = "node"
    node.vm.network "private_network", type: "dhcp"

    node.vm.network :forwarded_port, guest: 3646, host: 3646, auto_correct: true
    node.vm.network :forwarded_port, guest: 3200, host: 3200, auto_correct: true
    node.vm.network :forwarded_port, guest: 3500, host: 3500, auto_correct: true

    node.vm.network :forwarded_port, guest: 8500, host: 8500, auto_correct: true
    node.vm.network :forwarded_port, guest: 4646, host: 4646, auto_correct: true
    node.vm.network :forwarded_port, guest: 8080, host: 8080, auto_correct: true
    node.vm.network :forwarded_port, guest: 9998, host: 9998, auto_correct: true
    node.vm.network :forwarded_port, guest: 9999, host: 9999, auto_correct: true
    node.vm.network :forwarded_port, guest: 80,   host: 9080, auto_correct: true
    node.vm.network :forwarded_port, guest: 443,  host: 9443, auto_correct: true
    node.vm.post_up_message = <<~EOF
    Nomad has been provisioned and is available at the following web address:
    http://localhost:4646/ui/

    Nomad has Consul installed as well with web UI available at the following web address:
    http://localhost:8500/ui/
    EOF
  end

  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.synced_folder "/Users/justin/go/src/github.com/dantoml/nomad-plugin-firecracker/dist", "/opt/nomad/data/plugins", owner: "root", group: "root"
  config.vm.synced_folder "/Users/justin/Workspace/github.com/plasticine/ubuntu-firecracker/output", "/opt/nomad/img", owner: "root", group: "root"

  config.vm.provision "base", type: "shell", inline: $base, privileged: false
  config.vm.provision "docker", type: "shell", inline: $docker_install, privileged: false
  config.vm.provision "dnsmasq", type: "shell", inline: $dnsmasq, privileged: false

  config.vm.provision "download firecracker",type: "shell", inline: $firecracker_download, privileged: false
  config.vm.provision "download nomad",type: "shell", inline: $nomad_download, privileged: false
  config.vm.provision "download consul",type: "shell", inline: $consul_download, privileged: false

  config.vm.provision "install firecracker",type: "shell", inline: $firecracker_install, privileged: false
  config.vm.provision "install nomad",type: "shell", inline: $nomad_install, privileged: false
  config.vm.provision "install consul",type: "shell", inline: $consul_install, privileged: false

  config.vm.provision "configure nomad",type: "shell", inline: $nomad_config, privileged: false
  config.vm.provision "configure consul",type: "shell", inline: $consul_config, privileged: false

  config.vm.provision "consul-online-target", type: "shell", inline: $consul_online_target
  config.vm.provision "consul-online-service", type: "shell", inline: $consul_online_service
  config.vm.provision "consul-online-script",  type: "shell", inline: $consul_online_script

  config.vm.provision "enable consul", type: "shell", inline: "sudo systemctl enable consul.service"
  config.vm.provision "start consul", type: "shell", inline: "sudo systemctl start consul"
  config.vm.provision "sleep for Consul", type: "shell", inline: "sleep 5s"
  config.vm.provision "enable nomad", type: "shell", inline: "sudo systemctl enable nomad.service"
  config.vm.provision "start nomad", type: "shell", inline: "sudo systemctl start nomad"
end
