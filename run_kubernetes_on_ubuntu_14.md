# Run Kubernetes on Vagrant Virtual Machines

I have a 2012 late version Mac Mini with extended 16GB memory.  However, I cannot build Kubernetes on it even after installing boot2docer.  Details as described [here](https://github.com/wangkuiyi/kubernetes/issues/6).  So I am creating a Virtual Ubuntu 14 machine with 8GB memory and 4 cores using VirtualBox.

## Create a VM ssh-able from the Host

The installation of the VM is routinely.  One notable thing is that we need to add a Bridge VirtualBox network interface in addition to a NAT interface.  This kind of interface can get IP address from my router as a real network card does.  This allows me to ssh from my Mac OS X to the VM.  In order to configure this Bridge interface, we need to edit `/etc/network/interfaces` and add the following lines:

    auto eth1
    iface eth1 inet dhcp
    
Then we can bring it up using the following commands:

    sudo ifup eth1
    
We should see that eth1 (the Bridge network interface) retreiving IP address from the router and gets up.  Then we will be able to ssh from host to the VM.

## Install Prerequisits

### Before Installing Anything

    sudo apt-get update
    sudo apt-get upgrade
    
### Install Development Tools

    sudo apt-get install git make gcc
    
Note that building Kubernetes does not require `gcc`.  Indeed it does not even require Go compilers, because it builds using Docker container.  We need to install `gcc` because the building and installing of VirtualBox kernel module requires it.
    
### Vagrant

It is true that within Ubuntu 14 we can install Vagrant using `apt-get`. However, the version is not new enough.  So I downloaded Vagrant .deb files and install it:

    wget https://dl.bintray.com/mitchellh/vagrant/vagrant_1.6.5_x86_64.deb
    sudo dpkg -i ./vagrant_1.6.5_x86_64.deb 
    
To check the version of Vagrant, run

    vagrant --version    
    
### VirtualBox

To install the latest version of VirtualBox, I download and install the .deb file:

    wget http://download.virtualbox.org/virtualbox/4.3.20/virtualbox-4.3_4.3.20-96996~Ubuntu~raring_amd64.deb
    sudo dpkg -i ./virtualbox*.deb
   
This might prompt that there are some missing dependent packages.  To fix this, run:

    sudo apt-get -f install
    
To check the version of VirtualBox, run

    vboxmanage --version
    
Load kernel module:

    sudo /etc/init.d/vboxdrv setup    
    
This will recompile VirtualBox kernel modules.    
    
### Install net-tools

Install the net-tools package for your distribution for VirtualBox's private networks:

    sudo apt-get install net-tools
    


### Install Docker

This following command installs the latest release of Docker:

    curl -sSL https://get.docker.io/ubuntu/ | sudo sh
    
To check the version of Docker:

    docker --version    

The docker daemon always runs as the root user, and since Docker version 0.5.2, the docker daemon binds to a Unix socket instead of a TCP port. By default that Unix socket is owned by the user root, and so, by default, you can access it with sudo.

Starting in version 0.5.3, if you (or your Docker installer) create a Unix group called docker and add users to it, then the docker daemon will make the ownership of the Unix socket read/writable by the docker group when the daemon starts. The docker daemon must always run as the root user, but if you run the docker client as a user in the docker group then you don't need to add sudo to all the client commands. From Docker 0.9.0 you can use the -G flag to specify an alternative group.

Create a `docker` group:

    sudo groupadd docker

Add the current user into `docker` group:

    sudo usermod -a -G docker yi
    
where `yi` is a placeholder and should be your username.

Restart the Docker daemon.

    sudo service docker restart

Test running a Bash instance in a Ubuntu container:

    docker run -i -t ubuntu /bin/bash
        
## Build and Run Kubernetes

### Build
 
    git clone https://github.com/GoogleCloudPlatform/kubernetes.git
    cd kubernetes
    make release        
    
### Run

    cd kubernetes
    vagrant up
    
### Troubleshooting

#### VT-x is not valid

The `vagrant up` command complains the following error:

	==> master: Waiting for machine to boot. This may take a few minutes...
	The guest machine entered an invalid state while waiting for it
	to boot. Valid states are 'starting, running'. The machine is in the
	'poweroff' state. Please verify everything is configured
	properly and try again.
	
	If the provider you're using has a GUI that comes with it,
	it is often helpful to open that and watch the machine, since the
	GUI often has more helpful error messages than Vagrant can retrieve.
	For example, if you're using VirtualBox, run `vagrant up` while the
	VirtualBox GUI is open.

So I followed the suggestion and installed GUI on my VM:

    sudo apt-get install lxde-core xinit
    
This allows me to start X-window system and the LXDE desktop environment by typing:

    startx
    
Then I opened VirtualBox GUI.  There has been a virtual machine named `kubernetes_master_1417676144247_50960`.  

If I click the "start" button I can run this virtual machine. But it stops with error complaining no `VT-x is not available`.  This shows the reason that Vagrant cannot retreive but the VirtualBox GUI knows.

This error is due to that VirtualBox utilizes host CPUs to accelerate the running virtual machines by default.  However, as our host is itself a virtual machine and does not have sophisticated CPUs that supports the acceleration of virtual machines, this fails the booting of Kubernetes virtual machines.

#### Disable VT-x

The soluiton is to disable the utilization of CPU VT-x features.  This can be done by editing the Vagrantfile provided in Kubernetes source code.

The diff after editing is as follows:

	diff --git a/Vagrantfile b/Vagrantfile
	index 5be613e..f1ae4b6 100644
	--- a/Vagrantfile
	+++ b/Vagrantfile
	@@ -49,11 +49,20 @@ Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
	 
	     config.vm.provider :virtualbox do |v|
	       v.customize ["modifyvm", :id, "--memory", $vm_mem]
	-      v.customize ["modifyvm", :id, "--cpus", $vm_cpus]
	+      v.customize ["modifyvm", :id, "--cpus", "1"]
	 
	       # Use faster paravirtualized networking
	       v.customize ["modifyvm", :id, "--nictype1", "virtio"]
	       v.customize ["modifyvm", :id, "--nictype2", "virtio"]
	+
	+      # Disable hardware virtualization
	+      v.customize ["modifyvm", :id, "--hwvirtex", "off"]
	+      v.customize ["modifyvm", :id, "--hwvirtex", "off"]
	+      v.customize ["modifyvm", :id, "--nestedpaging", "off"]
	+      v.customize ["modifyvm", :id, "--largepages", "off"]
	+      v.customize ["modifyvm", :id, "--vtxvpid", "off"]
	+      v.customize ["modifyvm", :id, "--vtxux", "off"]
	+      v.customize ["modifyvm", :id, "--pae", "off"]
	     end
	   end

For more information about this editing, please refer to [this blog post](http://piotr.banaszkiewicz.org/blog/2012/06/10/vagrant-lack-of-hvirt/).

