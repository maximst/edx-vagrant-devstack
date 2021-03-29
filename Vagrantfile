Vagrant.require_version ">= 1.8.7"
unless Vagrant.has_plugin?("vagrant-vbguest")
  raise "Please install the vagrant-vbguest plugin by running `vagrant plugin install vagrant-vbguest`"
end

VAGRANTFILE_API_VERSION = "2"

MEMORY = 4096
CPU_COUNT = 2

# These are versioning variables in the roles. Each can be overridden, first
# with OPENEDX_RELEASE, and then with a specific environment variable of the
# same name but upper-cased.
VERSION_VARS = [
    'edx_platform_version',
    'configuration_version',
    'certs_version',
    'forum_version',
    'xqueue_version',
    'XQUEUE_VERSION',
    'demo_version',
    'NOTIFIER_VERSION',
    'ECOMMERCE_VERSION',
    'ECOMMERCE_WORKER_VERSION',
    'DISCOVERY_VERSION',
    'CONFIGURATION_VERSION',
    'EDX_PLATFORM_VERSION',
    'CERTS_VERSION',
    'FORUM_VERSION',
    'DEMO_VERSION',
    'THEMES_VERSION',
    'ACCOUNT_MFE_VERSION',
    'GRADEBOOK_MFE_VERSION',
    'PROFILE_MFE_VERSION',
]

MOUNT_DIRS = {
  :edx_platform => {:repo => "edx-platform", :local => "/edx/app/edxapp/edx-platform", :owner => "edxapp"},
  :themes => {:repo => "themes", :local => "/edx/app/edxapp/themes", :owner => "edxapp"},
  :forum => {:repo => "cs_comments_service", :local => "/edx/app/forum/cs_comments_service", :owner => "forum"},
  :ecommerce => {:repo => "ecommerce", :local => "/edx/app/ecommerce/ecommerce", :owner => "ecommerce"},
  :ecommerce_worker => {:repo => "ecommerce-worker", :local => "/edx/app/ecommerce_worker/ecommerce_worker", :owner => "ecommerce_worker"},
  :discovery => {:repo => "course-discovery", :local => "/edx/app/discovery/discovery", :owner => "discovery"},
  # This src directory won't have useful permissions. You can set them from the
  # vagrant user in the guest OS. "sudo chmod 0777 /edx/src" is useful.
  :src => {:repo => "src", :local => "/edx/src", :owner => "root"},
}
if ENV['VAGRANT_MOUNT_BASE']
  MOUNT_DIRS.each { |k, v| MOUNT_DIRS[k][:repo] = ENV['VAGRANT_MOUNT_BASE'] + "/" + MOUNT_DIRS[k][:repo] }
end

# map the name of the git branch that we use for a release
# to a name and a file path, which are used for retrieving
# a Vagrant box from the internet.
openedx_releases = {
  "open-release/juniper.1" => {
    :name => "edx-vagrant", :file => "juniper-rg.box",
  },
  "open-release/juniper.2" => {
    :name => "edx-vagrant", :file => "juniper-rg.box",
  },
  "open-release/juniper.3" => {
    :name => "edx-vagrant", :file => "juniper-rg.box",
  },
  "open-release/koa.1" => {
    :name => "edx-vagrant", :file => "edx-platform-koa.box",
  },
  "open-release/koa.2" => {
    :name => "edx-vagrant", :file => "edx-platform-koa.box",
  }
}
#openedx_releases.default = "open-release/koa.2"

openedx_release = ENV['OPENEDX_RELEASE'] || 'open-release/koa.2'
boxname = ENV['OPENEDX_BOXNAME']

# Build -e override lines for each overridable variable.
extra_vars_lines = ""
VERSION_VARS.each do |var|
  rel = ENV[var.upcase] || openedx_release
  if rel
    extra_vars_lines += "-e #{var}='#{rel}' \\\n"
  end
end

$script = <<SCRIPT
if [ ! -d /edx/app/edx_ansible ]; then
    echo "Error: Base box is missing provisioning scripts." 1>&2
    exit 1
fi

sudo chown vagrant:vagrant -R /home/vagrant
sudo chown edx-ansible:edx-ansible -R /edx/app/edx_ansible

export PYTHONUNBUFFERED=1
source /edx/app/edx_ansible/venvs/edx_ansible/bin/activate

cd /edx/app/ecommerce/ecommerce
sudo -u ecommerce git init
sudo -u ecommerce git remote add origin https://github.com/edx/ecommerce.git

cd /edx/app/discovery/discovery
sudo -u discovery git init
sudo -u discovery git remote add origin https://github.com/edx/course-discovery.git

cd /edx/app/edx_ansible/edx_ansible
sudo -u edx-ansible git checkout #{openedx_release}

cd /edx/app/edx_ansible/edx_ansible/playbooks

EXTRA_VARS="\
 -e@roles/edxapp/defaults/main.yml\
 -e@roles/common_vars/defaults/main.yml\
 -e EDXAPP_EDXAPP_SECRET_KEY=SET-ME-PLEASE\
 -e edxapp_user=edxapp\
 -e edxapp_data_dir=/edx/var/edxapp\
 -e common_web_group=www-data\
 -e DISCOVERY_NGINX_PORT=18381\
 -e ENTERPRISE_CATALOG_ENABLE_EXPERIMENTAL_DOCKER_SHIM=false\
 -e edx_django_service_enable_experimental_docker_shim=false\
"

EXTRA_VARS=$EXTRA_VARS" #{extra_vars_lines}"

CONFIG_VER="#{ENV['CONFIGURATION_VERSION'] || openedx_release || 'open-release/koa.2'}"

echo "#######################################################"
echo $EXTRA_VARS
echo "#######################################################"

ansible-playbook -i localhost, -c local run_role.yml -e role=edx_ansible -e configuration_version=$CONFIG_VER $EXTRA_VARS 

sudo -u edx-ansible cp vagrant-analytics.yml vagrant-devstack.yml
sudo -u edx-ansible sed -i '/- demo/d' vagrant-devstack.yml
sudo -u edx-ansible sed -i '/- analytics_api/d' vagrant-devstack.yml
sudo -u edx-ansible sed -i '/- insights/d' vagrant-devstack.yml
sudo -u edx-ansible sed -i '/- analytics_pipeline/d' vagrant-devstack.yml
sudo -u edx-ansible sed -i '13 a \\    DISCOVERY_URL_ROOT: "http://localhost:{{ DISCOVERY_NGINX_PORT }}"' /edx/app/edx_ansible/edx_ansible/playbooks/vagrant-devstack.yml
sudo -u edx-ansible sed -i '35 a \\    - discovery' /edx/app/edx_ansible/edx_ansible/playbooks/vagrant-devstack.yml
 
ansible-playbook -i localhost, -c local vagrant-devstack.yml -e configuration_version=$CONFIG_VER $EXTRA_VARS

sudo service supervisor stop
sudo systemctl disable supervisor

SCRIPT

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  boxfile = ""
  reldata = openedx_releases[openedx_release]
  if not boxname
    if Hash == reldata.class
      boxname = openedx_releases[openedx_release][:name]
      boxfile = openedx_releases[openedx_release].fetch(:file, "")
    else
      boxname = reldata
    end
  end
  if boxfile == ""
    boxfile = "#{boxname}.box"
  end

  # Creates an edX devstack VM from an official release
  config.vm.box     = boxname
  config.vm.box_url = "https://files.slack.com/files-pri/T042XMW6N-F01T8UNAKA4/download/#{boxfile}?pub_secret=6fb01420c2"
  config.vm.box_check_update = false

  config.vm.network :private_network, ip: "192.168.33.10"

  # If you want to run the box but don't need network ports, set VAGRANT_NO_PORTS=1.
  # This is useful if you want to run more than one box at once.
  if not ENV['VAGRANT_NO_PORTS']
    config.vm.network :forwarded_port, guest: 8000, host: 8000  # LMS
    config.vm.network :forwarded_port, guest: 8001, host: 8001  # Studio
    config.vm.network :forwarded_port, guest: 8002, host: 8002  # Ecommerce
    config.vm.network :forwarded_port, guest: 8003, host: 8003  # LMS for Bok Choy
    config.vm.network :forwarded_port, guest: 8004, host: 8004  # Discovery
    config.vm.network :forwarded_port, guest: 8031, host: 8031  # Studio for Bok Choy
    config.vm.network :forwarded_port, guest: 8120, host: 8120  # edX Notes Service
    config.vm.network :forwarded_port, guest: 8765, host: 8765
    config.vm.network :forwarded_port, guest: 9200, host: 9200  # Elasticsearch
    config.vm.network :forwarded_port, guest: 18080, host: 18080  # Forums
    config.vm.network :forwarded_port, guest: 8100, host: 8100  # Analytics Data API
    config.vm.network :forwarded_port, guest: 8110, host: 8110  # Insights
    config.vm.network :forwarded_port, guest: 9876, host: 9876  # ORA2 Karma tests
    config.vm.network :forwarded_port, guest: 50070, host: 50070  # HDFS Admin UI
    config.vm.network :forwarded_port, guest: 8088, host: 8088  # Hadoop Resource Manager
    config.vm.network :forwarded_port, guest: 18381, host: 18381    # Course discovery
  end
  config.ssh.insert_key = false

  config.vm.synced_folder  ".", "/vagrant", disabled: true

  # Enable X11 forwarding so we can interact with GUI applications
  if ENV['VAGRANT_X11']
    config.ssh.forward_x11 = true
  end

  if ENV['VAGRANT_USE_VBOXFS'] == 'true'
    MOUNT_DIRS.each { |k, v|
      config.vm.synced_folder v[:repo], v[:local], create: true, owner: v[:owner], group: "www-data"
    }
  else
    MOUNT_DIRS.each { |k, v|
      config.vm.synced_folder v[:repo], v[:local], create: true, nfs: true
    }
  end

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--memory", MEMORY.to_s]
    vb.customize ["modifyvm", :id, "--cpus", CPU_COUNT.to_s]

    # Virtio is faster, but the box needs to have support for it.  We didn't
    # have support in the boxes before Ficus.
    if !(boxname.include?("dogwood") || boxname.include?("eucalyptus"))
      vb.customize ['modifyvm', :id, '--nictype1', 'virtio']
    end
  end

  # Use vagrant-vbguest plugin to make sure Guest Additions are in sync
  config.vbguest.auto_reboot = true
  config.vbguest.auto_update = true

  # Assume that the base box has the edx_ansible role installed
  # We can then tell the Vagrant instance to update itself.
  config.vm.provision "shell", inline: $script
end
