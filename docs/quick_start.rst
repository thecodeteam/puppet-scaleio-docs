Quick Start
===========
The easiest way to start using the ScaleIO Puppet module is to delpoy it locally to a laptop using Vagrant and Virtual Box.  This method allows for a full deplyoment of Puppet, the ScaleIO Puppet module, and a working ScaleIO environment in five commands.

The default Vagrantfile includes one Puppet Master, and three ScaleIO nodes with mixed responsibilities. The Vagrantfile can be easily customized to deploy more nodes, and the module through the node classifier file (site.pp) is built to handle any configuration of nodes thereafter.

The module is currently intended to be used to configure a ScaleIO cluster from scratch for first-time MDM configurations. For running deployments, it can be used to define SDS items such as physical devices and volumes, and SDC items such as Volumes.

If you do not have Vagrant and Virtual Box installed, it is suggested that you either download them directly or use a package manager like http://brew.sh.  In addition you will need git installed or download the zip from https://github.com/emccode/vagrant-puppet-scaleio/archive/master.zip

Brew only works on Mac OS X.

::

  brew cask install virtualbox
  brew install vagrant
  brew install git

Get ScaleIO RPMs
----------------
In order to perform the install of the components on the VMs, you must have the ScaleIO RPMs.  These can be downloaded directly from the support.emc.com website.


Update ScaleIO Version
----------------------
There is one parameter that must be modified depending on your ScaleIO RPM version numbers.  There is a global parameter in the puppet/manifests/site.pp file that needs to be updated.

Check your RPM files and find the proper version number.  The following RPM file has a version of 1.30-426.0.  This is the SDC RPM, but is only being used as the example of identifying the version number.

::

  EMC-ScaleIO-sdc-1.30-426.0.el6.x86_64.rpm


Plug that version into the puppet/manifests/site.pp file.   Important Note:  If you are using the example site.pp files, make sure you update this version once you copy those files from puppet/manifests/examples to puppet/manifests/site.pp.

::

  $version = '1.30-426.0'



Vagrant Deploy
---------------
Issue the following commands to retrieve the proper repository with necessary files and issue an 'vagrant up' command.

::

  git clone https://github.com/emccode/vagrant-puppet-scaleio
  cd vagrant-puppet-scaleio
  mkdir -p puppet/modules/scaleio/files
  cp RHEL6/EMC-ScaleIO-*.rpm puppet/modules/scaleio/files
  cp ScaleIO_v1.30_GA/ScaleIO-GW\ \(IM+Rest-GW\)*.rpm puppet/modules/scaleio/files
  vagrant up
  vagrant status

Following this you should see the following Vagrant status report.

::

  Current machine states:

  puppetmaster              running (virtualbox)
  tb                        running (virtualbox)
  mdm1                      running (virtualbox)
  mdm2                      running (virtualbox)


There are three network adapters that are auto-configured by default.  Under the covers the 192.168.50.0/24 subnet is used for isolated ScaleIO communication.


==========================   ====================

Hostname                     IP

--------------------------   --------------------
tb                           192.168.50.11
mdm1						             192.168.50.12
mdm2						             192.168.50.13
==========================   ====================


Vagrant Customization
---------------------
The Vagrantfile is responsible for creating the proper VMs, configuring them, and then running the respective Puppet commands to start the server or agents.  The Vagrantfile and the ScaleIO Puppet module are completely separate and can be used independently.  

Configure the Puppet Master details in the puppetmaster_nodes hash.

.. code-block:: ruby

  puppetmaster_nodes = { 
    'puppetmaster' => { 
      :ip => '192.168.50.9', :hostname => 'puppetmaster', :domain => 'scaleio.local', :memory => 512, :cpus => 1 
    }
  }

The scaleio_nodes hash holds the configuration of the ScaleIO nodes.  Three nodes are needed at a minimum for ScaleIO to be configured.  Here you can see that we have a Tie-Breaker and two MDMs.  These nodes are all multi-role where they are also serving as servers and clients for data services.  The ScaleIO Puppet module determines these component roles.

.. code-block:: ruby

  scaleio_nodes = { 
    'tb' => { :ip => '192.168.50.11', :hostname => 'tb', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
    'mdm1' => { :ip => '192.168.50.12', :hostname => 'mdm1', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
    'mdm2' => { :ip => '192.168.50.13', :hostname => 'mdm2', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
  }


Vagrant Runtime
---------------
Vagrant status can be seen if you run 'vagrant status' from the directory with the Vagrantfile.  Before modifying the Vagrantfile make sure that you destroy the existing ScaleIO VMs 'vagrant destroy'.  

At any point you can add nodes to the Vagrant file scaleio_nodes hash followed by a 'vagrant up' command.  See Advanced Usage for more details.


Verify Node Deploy
-----------
You can verify at any time that the nodes have been configured using Puppet and ScaleIO commands.

From the node you can run a Puppet Agent command.  The following command will run the agent again and verify all configuration is correct.

::

  vagrant ssh node
  sudo puppet agent -t

From an MDM node you can run ScaleIO commands.  The following command will return general information.

::

  vagrant ssh mdm2
  sudo /bin/scli --login --username admin --password 'Scaleio123' --mdm_ip 192.168.50.12
  sudo /bin/scli --query_all --mdm_ip 192.168.50.12

Use this command for a verbose listing of commands.

::
  
  sudo /bin/scli --all --help



Puppet Node Redeploy
--------------------
If you have configured a node prior and want to redeploy, make sure that you remove the certificate for the old node from the Puppet Master.  Since ScaleIO requires that you install two MDM's and a Tie Breaker as a base install, do not try and redeploy these nodes.  This configuration must be done manually to get these nodes back online.  Redeploys of other node types do not have the same ramifications.

::
  
  vagrant ssh puppetmaster
  sudo puppet cert clean node.scaleio.local

If the node still exists you also must remove the certificate from the node.

::
  
  vagrant ssh node
  sudo find / -name node.scaleio.local.cer -delete

