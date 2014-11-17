Graphical UI
============
See the ScaleIO documentation for more details on loading the GUI.  

Untar the RPM
-------------
The GUI requires that you download Java JRE to the system that is going to run the GUI.  If you have brew installed you can leverage this to install via CLI.

::

  brew update && brew cask install java

Verify installation with the following command.

::
  
  which java

For Linux or OS X platforms, locate the GUI RPM named EMC-ScaleIO-gui-1.30-426.0.noarch.rpm.  This RPM has a shell script and a JAR file.  You can get the contents of the RPM without running an RPM command by using tar.

::
  
  tar -zxvf EMC-ScaleIO-gui-1.30-426.0.noarch.rpm
  ./opt/emc/scaleio/gui/run.sh 


Load the GUI
------------
If Java is in the path, then the GUI should load.  Enter admin for the username, and the password you configured in the site.pp file.  The IP address is the primary MDM ip.

::

  admin
  Scaleio123
  192.168.50.12

