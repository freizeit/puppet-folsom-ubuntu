# OpenStack folsom on ubuntu using puppet

Multi-node OpenStack `folsom` installation notes (on `ubuntu` using [puppet Openstack modules](https://github.com/puppetlabs/puppetlabs-openstack)).
Most of this was culled from the following article:
 [Openstack Folsom Deploy by Puppet on Ubuntu 12.04 HOWTO](http://edin.no-ip.com/blog/hswong3i/openstack-folsom-deploy-puppet-ubuntu-12-04-howto)

Please note also: in order for the puppet manifests to work all host names must be fully qualified (use .e.g. `.local` for a FQDN).

The machines we'll be using are as follows:

 - puppet master: `pm` at `172.24.0.10` (not part of the resulting cloud)
 - OpenStack controller node: `osc` at `172.24.0.11`
 - OpenStack compute node #1: `oscn1` at `172.24.0.12`
 - OpenStack compute node #2: `oscn2` at `172.24.0.13`

## Prepare the `puppetmaster` machine

 - install `ubuntu 12.04 LTS server`
 - add the puppetlabs repo
    <pre>
    wget http://apt.puppetlabs.com/puppetlabs-release-precise.deb
    sudo dpkg -i puppetlabs-release-precise.deb
    sudo apt-get update
    </pre>
 - install `puppetmaster` and `git`
    <pre>
    sudo apt-get install puppetmaster git
    </pre>
 - install `puppetlabs-openstack`
    <pre>
    git clone https://github.com/puppetlabs/puppetlabs-openstack
    (cd puppetlabs-openstack/ && git checkout folsom)
    sudo cp -r puppetlabs-openstack /etc/puppet/modules/openstack
    (cd puppetlabs-openstack && sudo rake modules:clone)
    </pre>
 - give the puppet master machine a fixed IP i.e. include the following
   stanza in `/etc/network/interfaces`:
   <pre>
    auto eth1
    iface eth1 inet static
      address 172.24.0.10
      netmask 255.255.0.0
      network 172.24.0.0
      broadcast 172.24.255.255
      gateway 172.24.0.1
      dns-nameservers 172.24.0.1
   </pre>
 - add the OpenStack controller and compute nodes to `/etc/hosts`
   <pre>
   172.24.0.11  osc.FQDN pm 
   172.24.0.12  oscn1.FQDN oscn1
   172.24.0.13  oscn2.FQDN oscn2
   </pre>
 - copy the the example site manifest into place
   <pre>
   sudo cp /etc/puppet/modules/openstack/examples/site.pp /etc/puppet/manifests/
   </pre>
 - apply [this patch](http://pastebin.ubuntu.com/1478229/)


## Prepare the `openstack_controller` machine
 - install `ubuntu 12.04 LTS server`
 - the `/etc/network/interfaces` file should look as follows:
    <pre>
    auto lo
    iface lo inet loopback
    auto eth0
    iface eth0 inet static
      address 172.24.0.11
      netmask 255.255.0.0
      network 172.24.0.0
      broadcast 172.24.255.255
      gateway 172.24.0.1
      dns-nameservers 172.24.0.1
    auto eth1
    iface eth1 inet manual
      up ifconfig eth1 up
    </pre>
 - add the puppetmaster machine to `/etc/hosts` like so
   <pre>
   127.0.0.1  osc.FQDN osc
   172.24.0.10  pm.FQDN pm 
   </pre>
 - name the `openstack_controller` machine in `/etc/hostname` like so
   <pre>
   osc.FQDN
   </pre>
 - add the puppetlabs repo
    <pre>
    wget http://apt.puppetlabs.com/puppetlabs-release-precise.deb
    sudo dpkg -i puppetlabs-release-precise.deb
    sudo apt-get update
    </pre>
 - install `puppet`
    <pre>
    sudo apt-get install puppet
    </pre>
 - configure the puppet agent as follows (in `/etc/puppet/puppet.conf`)
   <pre>
   [agent] 
   server=pm.FQDN
   certname=openstack_controller 
   </pre>
 - add the ubuntu/folsom repository:
   <pre>
   cat >> /etc/apt/sources.list <<-EOF
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/folsom main
   EOF
   aptitude install -y ubuntu-cloud-keyring
   aptitude update
   </pre>
 - invoke the puppet agent
   <pre>
   sudo puppet agent -vt
   </pre>
 - switch to the pupppet master console and sign the agent's certificate
   <pre>
   sudo puppet cert list
   sudo puppet cert sign osc.FQDN
   </pre>
   The step above is only necessary after the very first `puppet agent` call
 - re-run the puppet agent
   <pre>
   sudo puppet agent -vt
   </pre>

## Prepare the `openstack_compute` machine
 - install `ubuntu 12.04 LTS server`
 - the `/etc/network/interfaces` file should look as follows:
    <pre>
    auto lo
    iface lo inet loopback
    auto eth0
    iface eth0 inet static
      address 172.24.0.12
      netmask 255.255.0.0
      network 172.24.0.0
      broadcast 172.24.255.255
      gateway 172.24.0.1
      dns-nameservers 172.24.0.1
    auto eth1
    iface eth1 inet manual
      up ifconfig eth1 up
    </pre>
 - add the puppetmaster machine to `/etc/hosts` like so
   <pre>
   127.0.0.1  oscn1.FQDN oscn1
   172.24.0.10  pm.FQDN pm 
   </pre>
 - name the `openstack_compute` machine in `/etc/hostname` like so
   <pre>
   oscn1.FQDN
   </pre>
 - add the puppetlabs repo
    <pre>
    wget http://apt.puppetlabs.com/puppetlabs-release-precise.deb
    sudo dpkg -i puppetlabs-release-precise.deb
    sudo apt-get update
    </pre>
 - install `puppet`
    <pre>
    sudo apt-get install puppet
    </pre>
 - configure the puppet agent as follows (in `/etc/puppet/puppet.conf`)
   <pre>
   [agent] 
   server=pm.FQDN
   # omit certname, machine name will be assumed automatically
   </pre>
 - add the ubuntu/folsom repository:
   <pre>
   cat >> /etc/apt/sources.list <<-EOF
   deb http://ubuntu-cloud.archive.canonical.com/ubuntu precise-updates/folsom main
   EOF
   aptitude install -y ubuntu-cloud-keyring
   aptitude update
   </pre>
 - invoke the puppet agent and go for a coffee break:
   <pre>
   sudo puppet agent -vt
   </pre>


## Prepare the second `openstack_compute` machine
Repeat the `openstack_compute` setup for the second worker machine (`oscn2`) with the appropriate IP (`172.24.0.13`).

## Add an (ubuntu server) image to the cloud
In order to be able to start intances on the newly installed cloud we'll need an image. Please add it by running the following commands on the controller node:
   <pre>
   sudo su -
   source openrc
   wget http://uec-images.ubuntu.com/precise/current/precise-server-cloudimg-amd64-disk1.img
   glance image-create --name ubuntu.12.04.server --disk-format=qcow2 --container-format=bare --file precise-server-cloudimg-amd64-disk1.img
   </pre>

Please [see this page](http://docs.openstack.org/essex/openstack-compute/admin/content/starting-images.html) for a list of images.
