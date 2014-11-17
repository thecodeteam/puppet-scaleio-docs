Parameters
==========
There are a limited set of parameters that can be configured within the Puppet node classifier file.  This file (site.pp) can be used as is, or can be switched out for any one of Puppet's External Node Classifiers (ENC) for more dynamic functionality.  

The node classifier file (site.pp) is the file that a Puppet server uses to determine what a node configuration should be when it checks in.  This file matches nodes for their FQDN.  

Below we have broken up the (site.pp) file which can serve as a working example.


Global Parameters
-----------------
A global parameter gets set in the following section.  These parameters can also be set individually per node if desired.  These variable names are used later on in the node configuration section.


==========================   ====================

Parameter                    Description

--------------------------   --------------------

version (install required)   ScaleIO RPM version numbers that should be installed or upgraded to
sds_network (optional)       Network address that SDS should communicate on
mdm_fqdn (optional)          Can be used to define node classifications by name
mdm_ip (config required)     Array of MDM IP addresses as ['Primary','Secondary']
tb_fqdn                      Can be used to define Tie-Breaker classifications by name
tb_ip                        Tie-Breaker IP that is required if the ScaleIO cluster is not installed yet
cluster_name                 Cluster name that should be configured
enable_cluster_mode          Boolean that determines whether Cluster Mode is on or off
password                     New password to be set on the MDM during initial install
gw_password                  Password for the Gateway REST interface
==========================   ====================



.. code-block:: ruby

  $version = '1.30-426.0'
  $mdm_fqdn = ['mdm1.scaleio.local','mdm2.scaleio.local']
  $mdm_ip = ['192.168.50.12','192.168.50.13']
  $tb_fqdn = 'tb.scaleio.local'
  $tb_ip = '192.168.50.11'
  $cluster_name = "cluster1"
  $enable_cluster_mode = true
  $password = 'Scaleio123'
  $gw_password= 'Scaleio123'


SDS Devices
-------------------
The SDS Devices represent the block storage devices on an SDS node.  As part of the parameters you also specify which protection domain an SDS node takes part in.  The devices themselves can be block devices (partitions) that have a pre-determined size or files on file systems that will be truncated to create the sparse space.


This is an example of a hash that represents a node.

.. code-block:: ruby

  'FQDN' => {
    'ip' => 'IP of NODE',
    'protection_domain' => 'Protection Domain', 
    'devices' => {
      'Device Path1' => {  'size' => '100GB', 
                                        'storage_pool' => 'Name of Storage Pool'
                                      },
      'Device Path2' => {  'size' => '100GB', 
                                        'storage_pool' => 'Name of Storage Pool'
                                      },
    }

This are working hashes that represents nodes.

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
  }


SDC Volumes
-----------
The volumes section represents the actual consumable device that a client has available to carve space from.  Here you specify the Protection Domain and Storage Pool.  You can also specify any abritrary size in GB.  The sdc_ip represents an array of SDC IPs that will have access to the volume.

.. code-block:: ruby

  $sio_sdc_volume = {
    'volume1' => { 'size_gb' => 8, 
    'protection_domain' => 'protection_domain1', 
    'storage_pool' => 'capacity',
    'sdc_ip' => [
        '192.168.50.11',
        '192.168.50.12',
        '192.168.50.13',
      ] 
    }
  }


Call-Home
---------
The Call-Home section is used to configure the Call-Home service.

.. code-block:: ruby

  $callhome_cfg = {
    'email_to' => "emailto@address.com",
    'email_from' => "emailfrom@address.com",
    'username' => "monitor_username",
    'password' => "monitor_password",
    'customer' => "customer_name",
    'smtp_host' => "smtp_host",
    'smtp_port' => "smtp_port",
    'smtp_user' => "smtp_user",
    'smtp_password' => "smtp_password",
    'severity' => "error",
  }


Node Configuration
------------------
This is the node configuration.  With each node statement there is a regular expression that determines what configuration is applied to nodes with specific FQDNs.  The required configuration parameters differ depending on the node type.  Since the MDM and TB nodes specified here have mixed components there are more variables specified that needed for those node types.

See the Global Parameters section for details of other parameters listed below.  The components parameter is the only one that specifies something unique per node.

==========================   ====================

Parameter                    Description

--------------------------   --------------------

components                   An array of the components that can be installed (tb,sds,sdc,gw,lia,callhome)
==========================   ====================

Notice that the nodes are using a Regular Expression (/tb/) which matches any node that checks in with TB in the name.  The nodes can be duplicated and completely customizable based on site naming preferences.  Notice the components line where we specify (tb,sds,sdc,gw) meaning the node has is multi-role as a Tie-Breaker, SDS, SDC, and Gateway.

.. code-block:: ruby

node /tb/ {
  class {'scaleio::params':
        password => $password,
        version => $version,
        mdm_ip => $mdm_ip,
        tb_ip => $tb_ip,
        callhome_cfg => $callhome_cfg,
        sio_sds_device => $sio_sds_device,
        sds_ssd_env_flag => true,
        components => ['tb','sds','sdc'],
  }
  include scaleio
}

node /mdm/ {
  class {'scaleio::params':
        password => $password,
        version => $version,
        mdm_ip => $mdm_ip,
        tb_ip => $tb_ip,
        cluster_name => $cluster_name,
        sio_sds_device => $sio_sds_device,
        sio_sdc_volume => $sio_sdc_volume,
        callhome_cfg => $callhome_cfg,
        components => ['mdm','sds','sdc','callhome'],
  }
  include scaleio
}

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

node /sdc/ {
  class {'scaleio::params':
        password => $password,
        version => $version,
        mdm_ip => $mdm_ip,
        components => ['sdc'],
  }
  include scaleio
}

node /gw/ {
  class {'scaleio::params':
        gw_password => $gw_password,
        version => $version,
        mdm_ip => $mdm_ip,
        components => ['gw'],
  }
  include scaleio
}

