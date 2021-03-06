# puppet razor installation

`puppet` `razor` installation notes (on `ubuntu` using the [puppet razor module](http://puppetlabs.com/blog/puppet-razor-module/)).
Most of this was culled from the following article:
 [How to get started with Razor and Puppet – Part 1](http://purevirtual.eu/2012/07/02/how-to-get-started-with-razor-and-puppet-part-1/)

Please note also: in order for the puppet manifests to work all host names must be fully qualified (use .e.g. `.local` for a FQDN).

The machines we will be using are as follows:

 - razor/puppet master: `puppet` at `172.25.1.9` (not part of the resulting cloud)
 - OpenStack controller node: `osc` at `172.25.1.10`
 - OpenStack compute node #1: `oscn1` at `172.25.1.11`
 - OpenStack compute node #2: `oscn2` at `172.25.1.12`

## Prepare the `razor` machine

 - install `ubuntu 12.04 LTS server`
 - give the puppet master machine a fixed IP i.e. include the following
   stanza in `/etc/network/interfaces`:
   <pre>
    auto eth1
    iface eth1 inet static
      address 172.25.1.9
      netmask 255.255.255.0
      network 172.25.1.0
      broadcast 172.25.1.255
      gateway 172.25.1.1
      dns-nameservers 172.25.1.1
   </pre>
   in `/etc/hosts`:
   <pre>
    127.0.0.1       localhost
    127.0.1.1       puppet.FQDN puppet
   </pre>
   in `/etc/hostname`:
   <pre>
    puppet.FQDN
   </pre>
 - turn off the DHCP server for the `172.25.1.0` network (if you have one)
 - reboot the machine for the changes above to take effect
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
 - install `puppetlabs-razor`
    <pre>
    sudo puppet module install puppetlabs-razor
    sudo puppet apply /etc/puppet/modules/razor/tests/init.pp --verbose
    </pre>
 - install `dnsmasq` as follows:
    <pre>
    sudo puppet module install saz-dnsmasq
    sudo puppet apply /etc/puppet/modules/dnsmasq/tests/init.pp --verbose
    </pre>
 - create a file `razor-dnsmasq-config.pp` with the following content:
    <pre>
    dnsmasq::conf { 'razor-dnsmasq-config':
    ensure  => present,
          content => "dhcp-range=172.25.1.100,172.25.1.150,12h\ndhcp-boot=pxelinux.0\ndhcp-option=3,172.25.1.1\ndhcp-option=6,8.8.8.8",
}
    </pre>
 - and run puppet to allow the configuration to take effect
    <pre>
    sudo puppet apply razor-dnsmasq-config.pp
    </pre>
 - reboot one more time
 - last but not least make sure the razor daemon is up and running
    <pre>
    $ sudo /opt/razor/bin/razor_daemon.rb status
    razor_daemon: running [pid 9630]
    </pre>
    If it is not running please start it as follows:
    <pre>
    sudo /opt/razor/bin/razor_daemon.rb start
    </pre>

## booting the `razor` micro-kernel

Now boot up a machine that is attached to the `172.25.1.0` network in `PXE` mode and watch the magic happen :-)
Once it comes up you can view its properties on the `razor` machine as follows:
<pre>
$ sudo razor -n node
Discovered Nodes
        UUID           Last Checkin  Status                 Tags                  
63EC6bm1fE35UZEektJjh  37 sec        A       [memsize_1GiB,nics_2,kvm_vm,cpus_2]
</pre>
and
<pre>
$ sudo razor -n node 63EC6bm1fE35UZEektJjh
 UUID =>  63EC6bm1fE35UZEektJjh
 Last Checkin =>  01-10-13 14:57:14
 Status =>  active
 Tags =>  [memsize_1GiB,nics_2,kvm_vm,cpus_2]
 Hardware IDs =>  [525400E31381]
</pre>

## setting up an ubuntu server image

Booting the micro-kernel is only a first step. Let us setup `razor` so it can serve up a real OS.
<pre>
wget http://releases.ubuntu.com/precise/ubuntu-12.04.1-server-amd64.iso
sudo razor image add -t os -p ./ubuntu-12.04.1-server-amd64.iso -n ubuntu_precise -v 12.04
</pre>
This should result in something like this:
<pre>
Attempting to add, please wait...

New image added successfully

Added Image:
 UUID =>  6jwiYNcuAboUkaXSlkANKX
 Type =>  OS Install
 ISO Filename =>  ubuntu-12.04.1-server-amd64.iso
 Path =>  /opt/razor/image/os/6jwiYNcuAboUkaXSlkANKX
 Status =>  Valid
 OS Name =>  ubuntu_precise
 OS Version =>  12.04
</pre>

Next we will add a `razor` model as follows:
<pre>
$ sudo razor model add -t ubuntu_precise -l install_precise -i 6jwiYNcuAboUkaXSlkANKX
--- Building Model (ubuntu_precise): 

Please enter local domain name (will be used in /etc/hosts file) (example: example.com) 
default: localdomain
(QUIT to cancel)
 > noops.co
Please enter root password (> 8 characters) (example: P@ssword!) 
default: test1234
(QUIT to cancel)
 > 
Please enter node hostname prefix (will append node number) (example: node) 
default: node
(QUIT to cancel)
 > osn1
Model created
 Label =>  install_precise
 Template =>  linux_deploy
 Description =>  Ubuntu Precise Model
 UUID =>  7O1KQtGts9iIitORt88gNh
 Image UUID =>  6jwiYNcuAboUkaXSlkANKX
</pre>
