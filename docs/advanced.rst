Advanced
========
In addition to using example vagrant file (Vagrantfile-default) that defines VMs and the Puppet node configuration file (site.pp-default), you can also customize these files to meet install requirements for most environments.

The Puppet Master server shares the puppet directory with the Vagrant host (OS X in some cases).  Editing files under the puppet directory on the Vagrant host or /etc/puppet on the Puppet Master server have the same effect.  But always remember to reset the Puppet Master service if editing these files.

Add SDS Nodes
-------------

Vagrantfile Updates
~~~~~~~~~~~~~~~~~~~
For example, if you are running the default configuration (3 mixed use nodes), you may want to add new dedicated data service nodes (SDS) to add more storage.  Notice the following scaleio_nodes hash where we add two extra hash keys named sds1 and sds2.  Notice also that we add the SDS nodes before the mdm nodes!  This is important since the mdm2 node is responsible for configuring the cluster and data services and expects that the nodes have already been deployed and configured by the Puppet Agents.

.. code-block:: ruby

  scaleio_nodes = { 
    'tb' => { :ip => '192.168.50.11', :hostname => 'tb', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
    'sds1' => { :ip => '192.168.50.14', :hostname => 'sds1', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
    'sds2' => { :ip => '192.168.50.15', :hostname => 'sds2', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
    'mdm1' => { :ip => '192.168.50.12', :hostname => 'mdm1', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
    'mdm2' => { :ip => '192.168.50.13', :hostname => 'mdm2', :domain => 'scaleio.local', :memory => 1024, :cpus => 1 },
  }

Site.pp Updates
~~~~~~~~~~~~~~~
In addition to this we will need to update the sio_sds_device hash.  This should include a couple of lines with the FQDN of the hosts as keys.  See sds1.scaleio.local and sds2.scaleio.local where we also define identical capacity amounts.  Ordering is not critical here.

There is a pre-configured file at path puppet/manifest/examples/site.pp-2sds that can be copied over puppet/manifest/examples/site.pp.

.. code-block:: ruby

	$sio_sds_device = {
	  'tb.scaleio.local' => {
	    'ip' => '192.168.50.11',
	    'protection_domain' => 'protection_domain1', 
	    'devices' => {
	      '/home/vagrant/sio_device1' => {  'size' => '100GB', 
	                                        'storage_pool' => 'capacity'
	                                      },
	    }
	  },
	  'mdm1.scaleio.local' => {
	    'ip' => '192.168.50.12',
	    'protection_domain' => 'protection_domain1',
	    'devices' => {
	      '/home/vagrant/sio_device1' => {  'size' => '100GB', 
	                                        'storage_pool' => 'capacity'
	                                      },
	    }
	  },
	  'mdm2.scaleio.local' => {
	    'ip' => '192.168.50.13',
	    'protection_domain' => 'protection_domain1',
	    'devices' => {
	      '/home/vagrant/sio_device1' => {  'size' => '100GB', 
	                                        'storage_pool' => 'capacity'
	                                      },
	    }
	  },
	  'sds1.scaleio.local' => {
	    'ip' => '192.168.50.14',
	    'protection_domain' => 'protection_domain1',
	    'devices' => {
	      '/home/vagrant/sio_device1' => {  'size' => '100GB',
	                                        'storage_pool' => 'capacity'
	                                      },
	    }
	  },
	  'sds2.scaleio.local' => {
	    'ip' => '192.168.50.15',
	    'protection_domain' => 'protection_domain1',
	    'devices' => {
	      '/home/vagrant/sio_device1' => {  'size' => '100GB',
	                                        'storage_pool' => 'capacity'
	                                      },
	    }
	  },
	}

There is no need to update the sio_sdc_volume hash since this only relates to Volumes that are consumed and advertised to specific clients.

The last thing to set is the node statement.  Here we specify a regular expression, but can be hardcoded to a specific FQDN of a node.  Following this we specify the minimum amount of parameters that must be passed in order to configure the node for its component type (sds).  Notice how we specify /sds/ which matches any node with SDS in the FQDN while applying global variables.

The site.pp-default file already has this section.  Notice also the sds_ssd_env_flag setting.  This sould be set to true if you want to optimize the SDS operating system for solid state drives before deploying the SDS services.

.. code-block:: ruby

	node /sds/ {
	  class {'scaleio::params':
	        password => $password,
	        version => $version,
	        mdm_ip => $mdm_ip,
	        sio_sds_device => $sio_sds_device,
	        sds_ssd_env_flag => true,
	        components => ['sds'],
	  }
	  include scaleio
	}

Restart Puppet Master
~~~~~~~~~~~~~~~~~~~~~
The next step is to reset the Puppet Master service.  This is only necessary if you are going to add nodes to a running environment.  If you are going to start from scratch with the new configuration, then you do not need to restart the Puppet Master service.

::

  vagrant ssh puppetmaster
  sudo /etc/init.d/puppetmaster restart

Vagrant Deploy
~~~~~~~~~~~~~~
With Puppet configured, you are now ready to deploy the new Vagrant node.  If you want to start from scratch with your new configuration then you can do the following.

::

  vagrant destroy -f
  vagrant up

Otherwise, a simple 'vagrant up' command will find the missing nodes and deploy sds1 and sds2.

Force MDM2 to Check-In
~~~~~~~~~~~~~~~~~~~~~~
Changes to SDCs, SDSs, Volumes, or other data service configurations require that the MDM2 Puppet Agent runs after the Puppet Agents on the respective nodes run.  

::
  
  vagrant ssh mdm2
  sudo puppet agent -t

This will complete the configuration of the new SDS nodes.

Add New Devices to SDS
-----------------------
You can leverage this module to add new devices local to the SDS nodes to be consumed by the SDS service.  Consult ScaleIO documentation for the requirements for new storage to SDS devices.  Here we will show how to add same sized devices to all configured SDSs.

The following snippet shows the devices key with a single device (default) and two optional configuration parameters, size and storage_pool.  

.. code-block:: ruby

	'devices' => {
	'/home/vagrant/sio_device1' => {  
		'size' => '100GB',
		'storage_pool' => 'capacity'
	},  

Below is the full sio_sds_device hash with mutliple devices per SDS.


Site.pp Updates
~~~~~~~~~~~~~~~
There is a pre-configured file at path puppet/manifest/examples/site.pp-2devices that can be copied over puppet/manifest/examples/site.pp.

.. code-block:: ruby

	$sio_sds_device = {
	  'tb.scaleio.local' => {
	    'ip' => '192.168.50.11',
	    'protection_domain' => 'protection_domain1',
	    'devices' => {
	      '/home/vagrant/sio_device1' => {  'size' => '100GB',
	                                        'storage_pool' => 'capacity'
	                                      },
	      '/home/vagrant/sio_device2' => {  'size' => '100GB',
	                                        'storage_pool' => 'capacity'
	                                      },
	    }
	  },
	  'mdm1.scaleio.local' => {
	    'ip' => '192.168.50.12',
	    'protection_domain' => 'protection_domain1',
	    'devices' => {
	      '/home/vagrant/sio_device1' => {  'size' => '100GB',
	                                        'storage_pool' => 'capacity'
	                                      },
	      '/home/vagrant/sio_device2' => {  'size' => '100GB',
	                                        'storage_pool' => 'capacity'
	                                      },
	    }
	  },
	  'mdm2.scaleio.local' => {
	    'ip' => '192.168.50.13',
	    'protection_domain' => 'protection_domain1',
	    'devices' => {
	      '/home/vagrant/sio_device1' => {  'size' => '100GB',
	                                        'storage_pool' => 'capacity'
	                                      },
	      '/home/vagrant/sio_device2' => {  'size' => '100GB',
	                                        'storage_pool' => 'capacity'
	                                      },

Restart Puppet Master
~~~~~~~~~~~~~~~~~~~~~
The next step is to reset the Puppet Master service.  

::

  vagrant ssh puppetmaster
  sudo /etc/init.d/puppetmaster restart

Force TB, MDM1, and MDM2 to Check-In
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
This is possibly an optional statement.  Addition of new devices that must be created inside of the guest (truncate file), requires the Puppet Agent run on that node.  If it is an existing device inside of /dev/, then the Agent does not need to run and you can simply force the check in for mdm2.

Repeat this for tb, mdm1, and mdm2.  The mdm2 node must be done last in since it will configure the newly established devices.

::
  
  vagrant ssh node
  sudo puppet agent -t

This will complete the configuration of the Volume.

You should see the following at the end of the Puppet Agent run.


::

	Notice: /Stage[main]/Scaleio::Login/Exec[Normal Login Class]/returns: executed successfully
	Notice: mdm2.scaleio.local 
	Notice: /Stage[main]/Scaleio::Sds_first/Notify[mdm2.scaleio.local ]/message: defined 'message' as 'mdm2.scaleio.local '
	Notice: mdm1.scaleio.local 
	Notice: /Stage[main]/Scaleio::Sds_first/Notify[mdm1.scaleio.local ]/message: defined 'message' as 'mdm1.scaleio.local '
	Notice: tb.scaleio.local 
	Notice: /Stage[main]/Scaleio::Sds_first/Notify[tb.scaleio.local ]/message: defined 'message' as 'tb.scaleio.local '
	Notice: /Stage[main]/Scaleio::Sds/Exec[Add SDS mdm1.scaleio.local device /home/vagrant/sio_device2]/returns: executed successfully
	Notice: /Stage[main]/Scaleio::Sds/Exec[Add SDS mdm2.scaleio.local device /home/vagrant/sio_device2]/returns: executed successfully
	Notice: /Stage[main]/Scaleio::Sds/Exec[Add SDS tb.scaleio.local device /home/vagrant/sio_device2]/returns: executed successfully
	Notice: /Stage[main]/Scaleio::Sds_sleep/Exec[Add SDS mdm1.scaleio.local Sleep 30]/returns: executed successfully
	Notice: Finished catalog run in 36.98 seconds

Add Volume to SDC
------------------
You can leverage this module to manage the creation and mapping of Volumes to SDCs.  For this you there is a sio_sdc_volume hash in the site.pp file.  Notice how we added a key for volume2 specifying the size_gb, protection_domain, storage_pool, and which client SDC's to advertise the volume to.

Site.pp Updates
~~~~~~~~~~~~~~~
There is a pre-configured file at path puppet/manifest/examples/site.pp-volume2 that can be copied over puppet/manifest/examples/site.pp.


.. code-block:: ruby

  $sio_sdc_volume = {
    'volume1' => { 
      'size_gb' => 8,
      'protection_domain' => 'protection_domain1',
      'storage_pool' => 'capacity',
      'sdc_ip' => [
        '192.168.50.11',
        '192.168.50.12',
        '192.168.50.13',
      ]
    },
    'volume2' => { 
      'size_gb' => 16,
      'protection_domain' => 'protection_domain1',
      'storage_pool' => 'capacity',
      'sdc_ip' => [
        '192.168.50.11',
        '192.168.50.12',
        '192.168.50.13',
      ]
    }
  }


Restart Puppet Master
~~~~~~~~~~~~~~~~~~~~~
The next step is to reset the Puppet Master service.  

::

  vagrant ssh puppetmaster
  sudo /etc/init.d/puppetmaster restart

Force MDM2 to Check-In
~~~~~~~~~~~~~~~~~~~~~~
Changes to Volumes are performed from the mdm2 node.  

::
  
  vagrant ssh mdm2
  sudo puppet agent -t

This will complete the configuration of the Volume.

You should see the following at the end of the Puppet Agent run.

::

  Notice: /Stage[main]/Scaleio::Login/Exec[Normal Login Class]/returns: executed successfully
  Notice: /Stage[main]/Scaleio::Volume/Exec[Add Volume volume2]/returns: executed successfully
  Notice: /Stage[main]/Scaleio::Map_volume/Exec[Add Volume volume2 to SDC 192.168.50.12]/returns: executed successfully
  Notice: /Stage[main]/Scaleio::Map_volume/Exec[Add Volume volume2 to SDC 192.168.50.11]/returns: executed successfully
  Notice: /Stage[main]/Scaleio::Map_volume/Exec[Add Volume volume2 to SDC 192.168.50.13]/returns: executed successfully
  Notice: Finished catalog run in 4.74 seconds




