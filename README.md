##  Automation tool exercise ##

### Host system requirements

Host system is the hypervisor machine where we will be booting up the virtyual machine. In this exercise I have used ubuntu 14.04 as the host OS and KVM as the hypervisor. The host system should be up and running. You should be able to create virtual machine in it. I have used NAT enabled network for the virtual machine network. By default libvirt will create a NAT enabled network named ***default*** and start dhcp service in it. In the host system enable hardware virtualization support from the BIOS.

Run kvm-ok to see the virtualization support. Below output is desired

    #kvm-ok
    INFO: /dev/kvm exists
    KVM acceleration can be used    

If the host system is also a virtual machine then performance will be significantly lower.

### Installing Host system

- Install Ubuntu 14.04
- Login to the host system as a priviledged user. In this tutorial I have used root user
- This user should be part of libvirtd group
- Install software as mentioned below (If your host system is behind a proxy then you need to set http_proxy environmental variable at this point)
Here I have used root user.

        #apt-get update && apt-get upgrade -y
    	#apt-get install libvirt-bin qemu-kvm xauth
    	#apt-get install virtinst virt-manager bridge-utils dnsmasq
    	#sudo adduser `id -un` libvirtd
- Run ***"virsh net-list --all"*** command to see the default network is started
- If not then you need to start it manually by running below commands

        #virsh net-start default
    	#virsh net-autostart default
		#virsh net-list --all
		 Name                 State      Autostart     Persistent
		----------------------------------------------------------
		 default              active     yes           yes

- By default the network will start a dhcp server with IPs in 192.168.122.0/24 subnet. In case your host system is already in the same subnet the above commands will fail and you need to change the DHCP range before starting the default network. Use below command to do the same

		#virsh net-edit default
This will open a XML file. Edit it and save it and then start the network again


### Prerequisites for the automation tool
- A running local webserver accessible from the host system without using proxy
- CentOS 6.5 OS files should be hosted in the webserver. If you are planning to use a public URL then your host system should be able
to connect to the internet directly
- A priviledged user having sudo access and also the user should be part of libvirtd group. I have used root user for doing the tasks.

### Automation tool
This tool will create a KVM virtual machine and install CentOS 6.5. It will then install and configure httpd service in the vm.


Copy exercise.tar.gz to a **world readable location** like /tmp. Extract it there.

- Directory structure

        exercise
    	├── centos.ks
    	├── configrc
    	├── create_vm.sh
    	├── modules
    	│   └── modules.tar.gz
    	└── tests
    		└── runtest.sh


centos.ks => kickstart file
configrc => config file for the tool
create_vm.sh => Program to do installation of vm and service installation and configuration inside vm
modules/modules.tar.gz => Contains the puppet modules that are used in httpd service installation and configuration
tests/runtest.sh => Runs the functional testings

- Setting up configrc file

Below configuration defines the kickstart URL and the installtion URL. Rest parameters in "Installation section" defines the property of the VM.

default_kickstart_url="http://10.102.231.50:8080/centos/centos.ks" #Url of the kickstart file
default_install_url="http://10.102.231.50:8080/centos/6/x86_64/" #Url of the OS installation files (Should copy the content of DVD ISO to this localtion in webserver)

third_party_puppet_modules => The puppet modules mentioned here as a comma separated list. These modules will be installed in the virtual machine. In this exercise I have installaed puppetlabs-stdlib and jproyo-git modules. Although jproyo-git is not used but it is useful for cloning git repositories.

An example puppet file to install git and clone a repository from master branch is as follows

    include git
    $branch = master
    git::repo{'repo_name':
     path   => '/usr/src/repo',
     source => 'https://github.com/apache/httpd.git',
     branch => $branch,
    }

default_virt_type => If the host system is  physical machine then set it as kvm. If the host system is a virtual machine then set it as qemu.

In the Service section we mention the parameters needed for httpd configuration. You can change the below mentioned configurations. If you delete one or more of these parameters then the default values will be used. The default values are mentioned in myapache puppet module included in modules/modules.tar.gz file.

    User=nilanjan 
    Group=nilanjan
    Listen=80
    DocumentRoot="/var/www/html"


- Setting up centos.ks kickstart file

url --url="http://10.102.231.50:8080/centos/6/x86_64/" => Set this URL appropriately. Should be same as default_install_url in configrc

The default %post section is as below:

%post
#If you need to use a proxy to reach internet then uncomment below
#export http_proxy=http://10.102.238.156:8080
#export https_proxy=http://10.102.238.156:8080
rpm -ivh http://yum.puppetlabs.com/puppetlabs-release-el-6.noarch.rpm
yum install -y puppet-server
#create SSH keys
ssh-keygen -t rsa -N ""  -f /root/.ssh/id_rsa
#Copy the id_rsa.pub file of the host system user here
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCkgPR1/OF5nG8oSbDtLDRR2hvwCKcGQx3IPr2qHxMpvehV7M+06Won7ohkTXpz2Lh/q7hbXYOSoT7L2XYOTA8oR3bjvAS5TfsWz9GpqiI3GBWPuqBpdXqTQ+W2LvxOuiqLG2i8x9vMXzhr6lcLle9sxf1tVOuRkpYlUFmegx6PB8oHQrIv+f780CukO/V/u+lsjkzIfd8mP5/lDxi4lyU5UdmtvkfgeizEOBfYiGlkFr/cgbaugeYdOmCYba2oq0kOSOHVQBNKHGHroL17vBVm7oXmP4vnvFWoi8Q7vlnY6cwiV6VJllSze760Ktrjwh0dIwvoq205TmPV9Y6r9JQj root@ubuntu" > /root/.ssh/authorized_keys

#Include proxy configuration in /etc/bashrc. If you need to use a proxy to reach internet then uncomment below
#cat >> /etc/bashrc << EOF
#export http_proxy=http://10.102.238.156:8080
#export https_proxy=http://10.102.238.156:8080
#EOF
%end
 
Here two things need to be taken care
1. If you need proxy to reach internet comment out the lines that sets it.
2. Copy id_rsa.pub file (of the hosts system user) content in the line echo "xxxx" > /root/.ssh/authorized_key. This is required for having password less ssh access to the vm from the host

Copy the kickstart file to your webserver and set the value of default_kickstart_url in configrc to pint to the kickstart file URL.

- Running the tool

    	#cd exercise
    	#./create_vm.sh --name=nostovm

This script will do everything -
1. Create a virtual machine
2. Install CentOS 6 in it 
3. Install httpd configure httpd and start httpd.
4. Create configuration file for test case running.

A successful execution of the script produces below output

    root@ubuntu:/tmp/exercise# ./create_vm.sh --name=nostovm
    Staring Installation of nostovm base virtual machine
    Formatting '/tmp/exercise/images/nostovm.qcow2', fmt=qcow2 size=10737418240 encryption=off cluster_size=65536 lazy_refcounts=off
    
    Starting install...
    Retrieving file vmlinuz...   | 7.9 MB 00:00 ...
    Retrieving file initrd.img...|  64 MB 00:00 ...
    Creating domain...   |0 B 00:00
    Domain installation still in progress. You can reconnect to
    the console to complete the installation process.
    ##############################################################################
    1) yes
    2) no
    Would you like to see the installation progress: 1
    ##############################################################################
    1) vnc
    2) virt-manager
    3) quit
    Input the method to connect to the console of the vm:  2
    Checking DISPLAY environmental variable
    ##############################################################################
    Installing nostovm:  ##################################
    Installation of nostovm finished
    
    Starting up nostovm
    Domain nostovm started
    
    Waiting for VM nostovm to boot
    Configuring VM nostovm
    myapache/
    myapache/templates/
    myapache/templates/index.html
    myapache/templates/httpd.conf.erb
    myapache/manifests/
    myapache/manifests/init.pp
    puppetlabs-stdlib
    Notice: Preparing to install into /etc/puppet/modules ...
    Notice: Downloading from https://forgeapi.puppetlabs.com ...
    Notice: Installing -- do not interrupt ...
    /etc/puppet/modules
    └── puppetlabs-stdlib (v4.3.2)
    jproyo-git
    Notice: Preparing to install into /etc/puppet/modules ...
    Notice: Downloading from https://forgeapi.puppetlabs.com ...
    Notice: Installing -- do not interrupt ...
    /etc/puppet/modules
    └── jproyo-git (v0.1.0)
    Warning: Config file /etc/puppet/hiera.yaml not found, using Hiera defaults
    Notice: Compiled catalog for localhost in environment production in 0.65 seconds
    Warning: The package type's allow_virtual parameter will be changing its default value from false to true in a future release. If you do not want to allow virtual packages, please explicitly set allow_virtual to false.
       (at /usr/lib/ruby/site_ruby/1.8/puppet/type.rb:816:in `set_default')
    Notice: /Stage[main]/Myapache/Group[nilanjan]/ensure: created
    Notice: /Stage[main]/Myapache/User[nilanjan]/ensure: created
    Notice: /Stage[main]/Myapache/Package[httpd]/ensure: created
    Notice: /Stage[main]/Myapache/File[/var/www/html]/owner: owner changed 'root' to 'nilanjan'
    Notice: /Stage[main]/Myapache/File[/var/www/html]/group: group changed 'root' to 'nilanjan'
    Notice: /Stage[main]/Myapache/File[/etc/httpd/conf/httpd.conf]/content: content changed '{md5}27a5c8d9e75351b08b8ca1171e8a0bbd' to '{md5}27f8080dc63b5f7a486b2802c682acf6'
    Notice: /Stage[main]/Myapache/Service[httpd]/ensure: ensure changed 'stopped' to 'running'
    Notice: The URL of the service is http://192.168.122.148:80
    Notice: /Stage[main]/Myapache/Notify[info]/message: defined 'message' as 'The URL of the service is http://192.168.122.148:80'
    Notice: /Stage[main]/Myapache/File[/var/www/html/index.html]/ensure: defined content as '{md5}750fcdd444878be1691768445cf303f6'
    Notice: Finished catalog run in 5.16 seconds
    IP address of nostovm is 192.168.122.148
    Generating test configuration file



This script supports many command line arguments. But you can set them in the configrc file as well. To list all availabale options run 
    	
	#./create_vm.sh -h

It provides function to open the vm console while installation. There are two options vnc and virt-manager. For virt-manager to open a window you need to have a X server running in your host system.

- Follwoing test cases are included

1. Check whether httpd is installed in the vm
2. Check whether the configuration changes has been made in the httpd.conf file of the vm
3. Check whether httpd is running in the vm
4. Check whether httpd is listening to specified port

Run the below script to execute test

    #cd exercise/tests
	#./runtest.sh

A successful execution will produce below output 

    root@ubuntu:/tmp/exercise# cd tests/
    root@ubuntu:/tmp/exercise/tests# ./runtest.sh
    Service installation => PASSED
    Service configuration => PASSED
    Service Running => PASSED
    Service listening at specified port => PASSED