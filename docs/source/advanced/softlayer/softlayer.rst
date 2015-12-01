Using xCAT in SoftLayer
=======================

Overview
--------

In this documentation, there are some useful cases to use xCAT to manage SoftLayer bare metal servers, when SoftLayer doesn't currently provide the provisioning or management options needed.

And there are some things unique about the SoftLayer environment that make it a little challenging to use xCAT. This document gives some tips about how to configure things so that xCAT can be used. For some of these procedures, this document shows both an automated procedure and an manual procedure.

Currently, this document has only been written and validated for managing bare metal servers (not CCI virtual machines) and for provisioning SLES and Redhat.

Setting Up the xCAT Management Node
-----------------------------------

Creating and Configuring xCAT Management Node
```````````````````````````````````````````````

Use the SoftLayer portal to request either a VM (CCI) or bare metal server to run the xCAT management node software on. At a minimum, your xCAT management node should have these specs: ::

    a CCI (VM)
    2 CPUs
    8GB of memory
    100GB disk

If you will be provisioning multiple physical servers simultaneously, have specific provisioning performance requirements, or will be using sysclone, you will need to increase the resources. A fully capable xCAT management node that will be able to handle just about all the provisioning requirements you will have is: ::

    a bare metal server
    8 CPUs
    32 GB of memory
    2x1GB NIC on the private (backend) network, bonded together
    200 GB SSD drive for /install
    2x500 GB RAID'ed together

Depending on your needs, you may choose to configure your management node somewhere between these 2 extremes to balance performance and cost.

To configure the management node, follow the 1st half of XCAT iDataPlex Cluster Quick Start to install and configure the xCAT management node. Install xCAT 2.8.5 or later. Stop before you get to the section "Node Definition and Discovery". SoftLayer uses Supermicro servers, not iDataPlex, but they are both x86_64 IPMI-controlled servers, so they are very similar from an xCAT stand point. You don't need to follow all of the steps in [TODO:XCAT_iDataPlex_Cluster_Quick_Start]. There is a summary of the steps from that document that you should perform:

   #. You can skip most of the section "Prepare the Management Node for xCAT Installation". The OS will already be installed. You don't need to disable SELinux or the firewall, xCAT will do that for you. The networks and nics will already be set up by SoftLayer, and will not be using DHCP. The time zone will already be set. You don't need to configure switches, because we will use another method for discovery. You can leave the hostname set the way it is (to the public NIC). The only affect this will have is the ``master``, ``domain``, and ``nameservers`` site attributes will need to be set to the private NIC after the xCAT software is installed.

   #. Add the management node's private NIC IP address and hostname to ``/etc/hosts``.

   #. Follow the whole section "Install xCAT on the Management Node", using either option 1 or 2, except that you don't need to make the required packages from the distro available because SoftLayer will already have the correct yum repos set up for centos/rhel.

   #. Change the ``master`` and ``nameservers`` site attributes to be the private NIC IP address. Change the ``domain`` site attribute to the domain you will use for cluster management network.

   #. In the ``Configure xCAT`` section, set up the networks table, passwd table, and DNS, but do not set up dhcp or conserver.


Requesting Bare Metal Servers
`````````````````````````````

Now use the SoftLayer portal to request the bare metal servers that should be managed by xCAT. Request that they be loaded with the CentOS 6.x or sles operating system. Either centos or sles needs to be on the node for ``pushinitrd`` to function correctly. It will be more convenient if you request that all of the servers be on the same private (backend) vlan as the xcat management node. But this is not a requirement, xcat will work across vlans.

In this section, xCAT management node is installed CentOS 6.x.

Downloading the xCAT-SoftLayer RPM
```````````````````````````````

The utilities useful for running xCAT in a SoftLayer environment have been gathered into an xCAT-SoftLayer RPM for convenience. This RPM is not installed by default when you install xCAT core, so you must explicitly install it now. Install it: ::

    yum install xCAT-SoftLayer-*.rpm

The xCAT-SoftLayer rpm requires the ``perl-ExtUtils-MakeMaker``, ``perl-CPAN``, ``perl-Test-Harness``, and ``perl-SOAP-Lite rpms``.

Setting Up the SoftLayer API
````````````````````````

Use the SoftLayer portal to get your API key using these directions.

#. Download the SoftLayer perl API, using git, to any directory (you may have to install git): ::

    cd /usr/local/lib
    git clone https://github.com/softlayer/softlayer-api-perl-client

#. Create a file called ``/root/.slconfig`` and put in it your SoftLayer userid, the API key, and the location of the SL perl API: ::

    # Config file used by the xcat cmd getslnodes
    userid = SL12345
    apikey = 1a2b3c4d5e6f1a2b3c4d5e6f1a2b3c4d5e6f1a2b3c4d5e6f
    apidir = /usr/loca/lib/softlayer-api-perl-client

   Note: this config file will be used by the xCAT utility ``getslnodes``.

#. The softlayer api perl client also needs the following perl modules: ``XML::Hash::LX``, ``CPAN::Meta::Requirements``, ``Class::Inspector``, ``IO::SessionData``, ``lib::abs``, ``Test::Simple``. For now, you need to download these to your management node from ``CPAN`` and build them using the instructions on ``CPAN``. ::

    cpan App::cpanminus    # hit enter (taking the default "yes") a bunch of times
    cpanm XML::Hash::LX

Getting the Bare Metal Nodes Information and Defining It to xCAT
````````````````````````````````````````````````````````````````

#. To query all of the SL bare metal servers available to this account and display the xCAT node attributes that should be set: ::

    getslnodes

#. To query a specific server or subset of servers: ::

    getslnodes <hostname>

   where ``<hostname>`` is the 1st part of one or more hostnames of the SL bare metal servers.

#. To create the xCAT node objects in the database, either copy/paste/run the commands output by the command above, or run: ::

    getslnodes | mkdef -z

#. If your xCAT management node is also a bare metal server, this will create a node definition in the xCAT db for it too, which is probably not what you want. xCAT does support having the management node in the db and using xCAT to maintain software and config files on it, but that is probably not your main goal here, and you could accidentally make changes to your management node that you might not intend. If you want to remove your mgmt node from the db: ::

    rmdef <mgmt-node>

#. Now add the nodes to the ``/etc/hosts`` file: ::

    makehosts

#. Set up name resolution for the nodes: 

    a. Make sure the public and private networks of the xCAT MN are defined: ::

        lsdef -t network -l

    b. Set the site.nameservers attribute to be the private IP address of the xCAT management node (should be the same as site.master), and set the site.forwarders attribute to be the SL name servers, and set site.domain attribute to the domain the bare metal nodes are using: ::

        chdef -t site nameservers=<xcat-mn-ip> forwarders=<SL-name-servers> domain=<bm-domain>

    c. Edit ``/etc/resolv.conf`` to point to the xCAT management private IP as the name server and use the domain above: ::

        search <domain>
        nameserver <private-mn-ip>

    d. Have xcat configure and start dns on the MN ::

        makedns -n     

    e. Since we will be using static IP addresses for the nodes, there is no need for DHCP. We recommend you stop dhcpd so it doesn't confuse any debugging situations: ::

        service dhcpd stop
        chdef -t site dhcpsetup=n

    f. If you are deploying RHEL onto your nodes set managedaddressmode to static: ::

        chdef -t site managedaddressmode=static

      ``Note``: currently this site setting doesn't work correctly with the SLES support for configuring the private NIC into bond0.

Configuring Console Access for the Nodes
----------------------------------------

If the provisioning of nodes doesn't work perfectly the 1st time, access to the console can be critical in figuring out and correcting the problem. There are 3 options for getting access to each nodes' console:

    #. Connect to the Node's BMC Web Interface via VNC: this involves running a VNC server on your xCAT mgmt node and using a browser to connect to the BMC and launching the console in another window. 

    #. Configure VPN and use the BMC Web Interface or IPMIView: this requires a Windows client (desktop or laptop) to establish a VPN from your client to your SoftLayer nodes' private network, and then using the SoftLayer tool IPMIView to open a console. We've had problems with this approach.

    #. Configure the console for Use With xCAT rcons: this involves figuring out the serial console port and speed for each node can configuring the BMC accordingly. Once this is set up correctly, it is the most convenient option because you can bring up any console simply by running rcons. This option is explained in Appendix A.

Choose the option that suits you, and follow the instructions below.

Option 1 - Connecting to the Node's BMC Web Interface via VNC
`````````````````````````````````````````````````````````````

Use the BMC web interface to open a video console to a specific node. You must 1st install VNC and firefox.

#. On the xCAT management node:

   Verify that serialport and serialspeed are not set for the nodes you want to install. (The installer, e.g. autoyast, typically can only display to one console, the serial console or video console, but not both. If these serial console attributes are set, the installer will display its progress to the serial console, which we are not setting up in this option.) ::

    lsdef <node> -i serialport,serialspeed

   Set up the epel repo (for CentOS or RHEL), and install VNC, fluxbox, and firefox: ::

    yum install tigervnc-server firefox java icedtea-web fluxbox metacity xterm xsetroot

   Start the VNC server: ::

    vncserver -geometry 1280x960 -AlwaysShared &

#. On your client machine connect to the VNC server: ::

    vncviewer <xcat-mn-public-ip>:1 &

#. From inside the VNC session:

   In the xterm in VNC, if you don't like twm, start the window manager you want and add it to .vnc/xstartup for the future. For example, metacity: ::

    metacity &
    sed -i s/twm/metacity/ /root/.vnc/xstartup

   List the BMC information for the node you want to open the console to: ::

    lsdef <node> -i bmc,bmcusername,bmcpassword

   Start firefox inside vnc:

       a. In firefox, enter the IP address of the BMC, and then login with the BMC username and password.

       b. Choose menu option Remote Control -> Console Redirection.

       c. Click on Launch Console and choose to open the file with Iced Tea Web (``/usr/bin/javaws.itweb``), instead of save the file.

       d. When the black window opens, hit enter and you should see either a login prompt or a shell prompt.

Option 2 - Configuring VPN and Use the BMC Web Interface or IPMIView
``````````````````````````````````````````````````````````````````

Setting up VPN from your laptop/desktop to your SoftLayer account gives you direct access to the private network of your SoftLayer servers, so for access to the node consoles this is an alternative to using VNC. There are a couple different ways to configure SSL VPN for SoftLayer:

#. If you run windows on your laptop/desktop, point your browser to ``http://vpn.softlayer.com/`` and log in with your SoftLayer VPN id. This id is separate from you SoftLayer id and you'll have to create it the 1st time.
#. You can download, install, and run the stand alone Array Networks VPN client following ``https://vpn.dal05.softlayer.com/prx/000/http/localhost/login/help.html#clients`` . This works for linux and macs as well.
#. You can also VPN to SoftLayer using PPTP or Cisco AnyConnect, it can only allow 1 userid from your SoftLayer account to do this.

Once you have a VPN connection to SoftLayer then there are a couple ways to get a console to a SoftLayer bare metal server:

    a. Point your laptop/desktop browser to the BMC IP address of the server you want a console for. Login and choose the Remote Console menu option.
    b. Download/install the IPMIView tool from ``http://downloads.service.softlayer.com/ipmi`` .
    c. To get a console for a SoftLayer virtual machine (CCI), login to the manager or control portal and run the KVM Console action for the CCI.

Optional: Setting Up Additional NICs and Routes for xCAT to Configure
-----------------------------------------------------------------

xCAT has features to automatically configure additional NICs and routes on the nodes when they are being installed. In a SoftLayer environment, this can be convenient because there are usually additional NICs (other than the install NIC) and special routes that are needed. This section gives an example for nodes that have an eth1 NIC that is connected to the public VLAN and should be configured as part of bond1 and made the default gateway.

Setting Up Additional NICs
````````````````````````

#. Make sure the networks for the public vlans are defined in the xcat database: ::

    lsdef -t network -l

   If they aren't, define them: ::

    mkdef -t network publicnet gateway=50.97.240.33 mask=255.255.255.240 mgtifname=eth1 net=50.97.240.32

   ``Note``: in the networks table, mgtifname means the NIC on the xcat mgmt node that directly connects to that vlan. If the xcat mgmt node is not directly connected to this vlan (it reaches it via a router), then set mgtifname to ``!remote!<nicname>`` and add ``!remote!`` for ``dhcpinterfaces`` in site table.

#. For each node, set the IP address and hostname suffix and the network for the public network. In this example, we assume the public NICs will be configured to be bonded together, which works even if one of the NICs (e.g. eth3) is down: ::

    chdef <node> nicips.eth1=50.2.3.4 nichostnamesuffixes.bond1=-pub

   ``Note``: In this example, the ``-pub`` will be added to the end of the node name to form the hostname of the eth1 IP address. If you also want a completely different hostname (that doesn't start with the node name), set nicaliases.eth1. If you have a lot of nodes and your IP addresses follow a regular pattern, you can set them all at once, using xCAT's support for regular expressions.

#. Add these new NICs to name resolution: ::

    makehosts <noderange>
    makedns <noderange>

   If you are running a version of xcat older than 2.8.5 and you have nodes that are on a private vlan that is different from the xcat mgmt node's private vlan, then you need to add the following line to the global "options" statement in /etc/named.conf so that the nodes on the other private vlans will be allowed to query the dns on the xcat mgmt node: ::

    allow-recursion { any; };

   After editing ``/etc/named.conf``, then run ``service named restart``. If you run ``makedns -n`` in the future, you will need to make this change to ``/etc/named.conf`` again, because it will be overwritten.

#. You would use the confignics postscript to configure eth1 at the end of the node provisioning. But since SoftLayer bare metal servers should have their NICs part of a bond, use the configbond postscript instead by adding it to the list of postscripts that should be run for these nodes: ::

    chdef <noderange> -p postscripts='configbond bond1 eth1@eth3'

   ``Note``: the ``-p`` flag adds the postscript to the end of the existing list.

   Test the setup on 1 node using updatenode: ::

    updatenode <node> -P 'configbond bond1 eth1@eth3'

   Set installnic to "mac" to select the install NIC by mac address instead of NIC name (e.g. eth0), because the NIC name can vary depending on what OS or initrd is booted: ::

    chdef <node> installnic=mac

Setting Up Routes
``````````````````

You can set up routes (both default gateway and more specific routes) to be configured on the nodes using the routes table, the routenames attribute and the setroute postscript. These 3 work together like this:

    a. Define routes in the routes table for any routes you will need on any of nodes and give them unique names;
    b. For each node set its routenames attribute to the routes you want that node to have;
    c. Add the setroute postscript to the postbootscripts attribute for all nodes.

#. If you want to set the default gateway of the nodes to go out to the internet, create a route entry that points to the gateway IP address that SoftLayer defines for the public vlan for this node: 
   ::
 
    mkdef -t route <route_name> gateway=198.11.206.1 ifname=bond1 mask=0.0.0.0 net=0.0.0.0

   ``Note``: ifname is the NIC that the node will use to reach this gateway.

   Add this route to the node definitions and add setroute to the postbootscripts list: ::

     chdef <noderange> -p routenames=<route_name>
     chdef <noderange> -p postbootscripts='setroute'

#. If you are setting the node's default gateway to the public NIC, you will want a specific route for the private VLANs if you have servers in more than 1 private vlan:

   Create a route in the route table: ::

    mkdef -t route <route_name> gateway=10.54.51.1 ifname=bond0 mask=255.0.0.0 net=10.0.0.0

   Add this route to the node definitions and add setroute to the postbootscripts list: ::

    chdef <noderange> -p routenames=<route_name>
    chdef <noderange> -p postbootscripts='setroute'

   If some of the nodes you are installing are on a different private vlan than the xcat mgmt node, you need to set those nodes' xcatmaster attribute to the ip addresss of the mgmt node. For example: ::

    chdef <noderange> xcatmaster=10.54.51.2

   Test one node using updatenode: ::

    updatenode <noderange> -P 'setroute'

Configure the Node to Install a New OS (Scripted Install)
---------------------------------------------------------

Because SoftLayer switches often respond to NIC state changes slowly (when the NICs are not bonded) and because bare metal nodes are often allocated on different vlans from the xCAT MN, it is necessary to use a different method for initiating the network installation of the node. (Normally, xCAT relies on PXE and DHCP broadcasts during the network installation process, which by default don't go across vlan boundaries.) The basic approach we will use is to copy to the node the kernel, initrd, and IP address that xCAT will use to install the node. After that, the xCAT node installation process will proceed like usual.

Using the xCAT scripted install method is covered more fully in [TODO:XCAT_iDataPlex_Cluster_Quick_Start]. Use this section here in this document as a supplement that is specific to the SoftLayer environment.

Prepare an Osimage to Use For the Nodes
```````````````````````````````````````

#. Create an OS image defintion on the xCAT MN that will be used to provision the node, following Option 1 in XCAT iDataPlex Cluster Quick Start#Installing Stateful Nodes.

#. Get the driver disk for the aacraid driver and add it to the osimage definition. This is necessary because many SoftLayer physical servers use that device, but that driver is not in the default initrd. For example: ::

    chdef -t osimage <osimagename> driverupdatesrc=dud:/install/drivers/sles11.3/x86_64/aacraid-driverdisk-1.2.1-30300-sled11-sp2+sles11-sp2.img

   ``Note``: the aacraid rpm has an unusual format, so you can't use that with xCAT.

#. Modify your osimage definition to use the provided autoyast template that uses a static IP for the install NIC, instead of the typical DHCP IP address that xCAT normally uses: ::

    chdef -t osimage <osimagename> template=/opt/xcat/share/xcat/install/sles/compute.sles11.softlayer.tmpl

   ``Note``: so far only a template for SLES has been provided.

   If desired, you can specify a specific partition layout. For example, create a file called ``/install/custom/my-partitions``, containing: ::

        <drive>
          <device>XCATPARTITIONHOOK</device>
          <initialize config:type="boolean">true</initialize>
          <use>all</use>
          <partitions config:type="list">
            <partition>
              <create config:type="boolean">true</create>
              <filesystem config:type="symbol">swap</filesystem>
              <format config:type="boolean">true</format>
              <mount>swap</mount>
              <mountby config:type="symbol">path</mountby>
              <partition_nr config:type="integer">1</partition_nr>
              <partition_type>primary</partition_type>
              <size>32G</size>
            </partition>
            <partition>
              <create config:type="boolean">true</create>
              <filesystem config:type="symbol">ext3</filesystem>
              <format config:type="boolean">true</format>
              <mount>/</mount>
              <mountby config:type="symbol">path</mountby>
              <partition_nr config:type="integer">2</partition_nr>
              <partition_type>primary</partition_type>
              <size>64G</size>
            </partition>
          </partitions>
        </drive>

   Then use this file in your osimage: ::

    chdef -t osimage <osimagename> partitionfile=/install/custom/my-partitions

Deploying Nodes With That Osimage
```````````````````````````````

#. If the nodes to be deployed are not already up, boot the nodes to the existing OS that is already on its hard disk: ::

    rsetboot <noderange> hd
    rpower <noderange> boot

#. Copy the xCAT management node's ssh public key to the nodes: ::

    lsdef <noderange> -ci usercomment   # note the pw of each node
    xdsh <node> -K       # enter node pw when prompted

   ``Note``: you can skip this step if when you originally requested the servers from the SoftLayer portal, you gave it the xCAT management node's public key to put on the servers.

#. Have xCAT generate the initrd, kernel, and kernel parameters for deploying the nodes: ::

    nodeset <noderange> osimage=<osimagename>

#. Use the xCAT script called pushinitrd to automatically push the initrd, kernel, kernel parameters, and static IP information to the nodes: ::

    pushinitrd <noderange>

#. If this is the 1st node installation you've done in this cluster, open a console to one of the nodes before starting the installation, so that you can follow the process and see any errors that might be displayed. (See the previous section for how to get a console.) If you already know that the installation process works, you don't need to open a console.

   Boot the nodes to start the installation process: ::

    rsetboot <noderange> hd
    xdsh <noderange> reboot

   ``Note``: For some physical server types in softlayer, the rsetboot command fails. You can still proceed without it, it just means you will have to wait for the nodes to time out waiting for DHCP. Do not use rpower to reboot the node in this situation because that does not give the nodes a chance to sync the file changes (that pushinitrd made) to disk.

#. Monitor the progress: ::

    watch nodestat <noderange>

   Later on, if you want to push updates to the nodes (without completely reinstalling them), you can use the updatenode command. With this command you can sync new config files to the nodes, install additional rpms, and run new postscripts.

Using Sysclone (SystemImager) to Clone Nodes
----------------------------------------------

You can also use xCAT's sysclone method of capturing an image and deploying it to nodes, instead of using the scripted install method of node deployment (which is described in the previous chapter). The sysclone method (which uses the open source tool SystemImager underneath) enables you to install 1 golden node using the xCAT scripted install method, then further configure it exactly how you want it, capture the image, and then deploy that exact image on many nodes. If even enables you to subsequently capture updates to the golden node and push out just those deltas to the other nodes, making the updates much faster.

Using sysclone is covered more fully in `Using System Clone to Deploy Diskful Node <Using System Clone to Deploy Diskful Node>`. Use this section here in this document as a supplement that is specific to the SoftLayer environment. These differences are needed because SoftLayer switches often respond to NIC state changes slowly (when the NICs are not bonded) and because bare metal nodes are often allocated on different vlans from the xCAT MN.

Prepare the xCAT Mgmt Node for Using Sysclone
``````````````````````````````````````````````

In this example, xCAT management node is installed sles.

These steps only need to be done once, to prepare the xCAT mgmt node for using sysclone.

#. Add the Adaptec aacraid device driver to the xCAT genesis initrd. Follow the example given in XCAT_iDataPlex_Advanced_Setup#Adding_Drivers_to_the_Genesis_Boot_Kernel . This is needed because many of the SoftLayer servers have this device.

#. Install SystemImager on the xCAT mgmt node. The systemimager rpms are in the xcat-dep tarball, so you should already have that configured as a zypper archive on your mgmt node. (TODO:See section XCAT_iDataPlex_Cluster_Quick_Start#Using_the_New_Sysclone_Deployment_Method and the sections before that for the full context.) ::

    zypper install systemimager-server

#. Start the rsync daemon for systemimager: ::

    service systemimager-server-rsyncd start
    chkconfig systemimager-server-rsyncd on

#. Make sure you have the xcat-dep rpms in an otherpkgs directory. For example: ::

    mkdir -p /install/post/otherpkgs/sles11.3/x86_64/xcat
    cd /install/post/otherpkgs/sles11.3/x86_64/xcat
    tar jxvf xcat-dep-*.tar.bz2

Capture the Image of a Golden Node
```````````````````````````````````

``Golden Node`` is a term that means the server that you will configure the way you want many of your nodes to be and then take a snapshot of that image. You can have more than one golden node, if you have different types of nodes in your cloud. Normally, you should keep your golden nodes around long term, so that you can go back to them, apply updates, and capture the deltas.

Follow these steps to prepare a golden node and then capture its image. Some of these steps are described in more detail in TODO:XCAT_iDataPlex_Cluster_Quick_Start#Option_2:_Installing_Stateful_Nodes_Using_Sysclone, including its 2 subsections "Install or Configure the Golden Client" and "Capture image from the Golden Client". You should read the details in these 2 subsections, but the summary of what you will need to do is here, plus some additional steps.

#. Install the operating system and other desired software on the golden node. This can be done manually, or using xCAT's scripted install method described in the previous section.

#. Install these rpms on the golden node: ``systemimager-common``, ``systemimager-client``, ``systemconfigurator``, ``perl-AppConfig``. This is most easily done by updating the osimage definition you use for the golden node and then using xCAT's updatenode command. On the mgmt node do: ::

    chdef -t osimage -o <osimage-name> otherpkglist=/opt/xcat/share/xcat/install/rh/sysclone.sles11.x86_64.otherpkgs.pkglist
    chdef -t osimage -o <osimage-name> -p otherpkgdir=/install/post/otherpkgs/sles11.3/x86_64
    updatenode <my-golden-cilent> -S

   If you are running a version of xCAT older than 2.8.5, one additional step that is necessary is to add the following lines to ``/etc/systemimager/updateclient.local.exclude`` on the golden node: ::

    # These are files/dirs that are created automatically on the node, either by SLES, or by xCAT.
    /boot/grub
    /etc/grub.conf
    /etc/hosts
    /etc/udev/rules.d/*
    /etc/modprobe.d/bond0.conf
    /etc/modprobe.d/bond1.conf
    /etc/ssh
    /etc/sysconfig/syslog
    /etc/syslog-ng/syslog-ng.conf
    /opt/xcat
    /root/.ssh
    /var/cache
    /var/lib/*
    /xcatpost

#. Capture the golden node image, by running this on the xCAT mgmt node: ::

     imgcapture <my-golden-client> -t sysclone -o <myimagename>

   This will rsync the golden node's file system to the xCAT mgmt node and put it under ``/install/sysclone/images/<image-name>``.

Deploy the Image to a New Bare Metal Server
```````````````````````````````````````````

Once the image has been captured, use these steps to deploy it to one or more nodes. All of these steps are performed on the xCAT mgmt node.

#. If the nodes to be deployed are not already up, boot the nodes to the existing OS that is already on its hard disk: ::

    rsetboot <noderange> hd
    rpower <noderange> boot

#. Copy the xCAT management node's ssh public key to the nodes: ::

    lsdef <noderange> -ci usercomment   # note the pw of each node
    xdsh <node> -K       # enter node pw when prompted

   ``Note``: you can skip this step if when you originally requested the servers from the SoftLayer portal, you gave it the xCAT management node's public key to put on the servers.

#. Have xCAT generate the initrd, kernel, and kernel parameters for deploying the nodes: ::

    nodeset <noderange> osimage=<captured-sysclone-image>

#. Use the xCAT script called pushinitrd to automatically push the initrd, kernel, kernel parameters, and static IP information to the nodes: ::

    pushinitrd <noderange>

#. Boot the nodes to start the installation process: ::

    rsetboot <noderange> hd
    xdsh <noderange> reboot

   ``Note``: For some physical server types in softlayer, the rsetboot command fails. You can still proceed without it, it just means you will have to wait for the nodes to time out waiting for DHCP. Do not use rpower to reboot the node in this situation because that does not give the nodes a chance to sync the file changes (that pushinitrd made) to disk.

#. Monitor the progress: ::

    watch nodestat <noderange>

Updating Nodes Later On
````````````````````````

If you need to make changes to the golden client (install new rpms, change config files, etc.), you can capture the changes and push them to the already cloned nodes. This process will only transfer the deltas, so it will be much faster than the original cloning.

    * Make changes to your golden node.

    * From the mgmt node, capture the image using the same command as before. Assuming <myimagename> is an existing image, this will only sync the changes to the image on the mgmt node. ::

       imgcapture <my-golden-client> -t sysclone -o <myimagename>

**If you are running xCAT 2.8.5 or later:**

    * For the nodes you want to update with this updated golden image: ::

       updatenode <noderange> -S

**If you are running xCAT 2.8.4 or older:**

    * For one of the nodes you want to update, do a dry run of the update to see which files will be updated: ::

       xdsh <node> -s 'si_updateclient --server <mgmtnode-ip> --dry-run --yes'

    * If it lists files/dirs that you don't think should be updated, you need to add them to the exclude list in 3 places: ::

       On the golden node: /etc/systemimager/updateclient.local.exclude
       On the mgmt node: /install/sysclone/images/<myimagename>/etc/systemimager/updateclient.local.exclude
       On all of the nodes to be updated: /etc/systemimager/updateclient.local.exclude

    * From the mgmt node, push the updates out to the other nodes: ::

       xdsh <noderange> -s 'si_updateclient --server <mgmtnode-ip> --yes'

    * Run mkinitrd on the nodes because each node needs an initrd that is appropriate for its hardware, not the initrd in the image that was just sync'd: ::

       xdsh <noderange> -s mkinitrd      # only valide for sles/suse, for red hat use dracut

If you want more information about the underlying SystemImager commands that xCAT uses, see the SystemImager user manual.

Appendix A - Configure the Console for Use With xCAT rcons
----------------------------------------------------------

It is possible to configure the SoftLayer node's BMCs to work with the xCAT rcons command. Once this is set up, the rcons command is a very convenient way to view the node's consoles. But initially setting it up is not simple. Basically, you need to configure conserver on the xCAT mgmt node, and then determine each node's serial console port and speed and configure the BMC accordingly.

To determine the console port number and speed that should be used, and to configure everything accordingly, follow this procedure. Because the different Supermicro server models that SoftLayer uses are not consistent about the COM port and speed, this process is currently a little bit trial and error.

**On the xCAT management node:**

    * Consider setting ``consoleondemand=yes``, which tells conserver to only connect to the console when the rcons command is run for that node. By default, conserver tries to connect to all consoles when it starts (and tries to reconnect if a console connection ever drops), so that it can constantly be logging the console output for reference later on. This is handy, but also means that conserver can fight with SoftLayer for the single console on each bmc. ::

       chdef -t site consoleondemand=yes

    * Pick one node to set up and get conserver configured: ::

       chdef <node> cons=ipmi
       makeconservercf <node>

    * For convenience, transfer the MN's ssh public key to the node: ::

       getslnodes <node>    # note the node's password
       xdsh <node> -K       # enter the password when prompted
       ssh <node> date      # verify that the date command runs on the node w/o being prompted for a pw

    * Then in a separate shell that you can leave open for a while, run: ::

       rcons <node>

**Now ssh to the node and do:**

    * Load the ipmi kernel module: ::

       yum install ipmitool      # if not already installed
       modprobe ipmi_devintf

    * Query the speed that the bmc is currently configured to, and query the COM ports that exist on this node: ::

       ipmitool sol info 1      # note the speed that the bmc is currently using
       dmesg|grep ttyS          # to see com ports avail (the deprecated msg is ok)

    * Use the screen command to try each COM port to see which one is use for SOL: ::

       yum install screen       # if not already installed
       screen /dev/ttyS1 115200   # try COM 2, use the speed the bmc is using

    * In the screen above, type some text and hit enter (you won't see the text echoed) and see if it comes out on rcons. If it does, that's the port. If it doesn't, try the next port. To get out of the screen, type ``ctrl-shift-a``,``shift-k``
    * Figure out what speed the bios is using by trial & error (try 19200, 57600 and reboot after each one and watch rcons)

    * Set the bmc to use the bios speed. This enables rcons to show output from both the bios portion of the booting and the booting of the OS: ::

       ipmitool sol set volatile-bit-rate 19.2 1     # use the speed the bios is using
       ipmitool sol set non-volatile-bit-rate 19.2 1

**Back on the xCAT MN:**

    * Set the console port and speed in the xCAT db (this is used by nodeset to set the node's kernel parameters), for example: ::

       chdef <node> serialport=2 serialspeed=19200

    * Reboot the node to test the console display: ::

       rsetboot <node> hd;     # set the next boot of the node to be the current OS on its local hard disk
       xdsh <node> reboot      # to test rcons

Appendix B - Manually Push the Network Installation Settings to the Nodes
-------------------------------------------------------------------------

If you want to manually copy the network installation files and settings to the nodes, instead of using the pushinitrd command, follow these steps:

    * Have xCAT generate the initrd, kernel, and kernel parameters: ::

       nodeset <node> osimage=sles11.2-x86_64-install-compute

    * Display the initrd, kernel, and kernel parameters: ::

       nodels <node> bootparams

    * Using the paths displayed in the nodels command above, and prefixing them with ``/tftpboot/``, copy the kernel and initrd to the /boot file system of the node. For example: ::

       scp /tftpboot/xcat/osimage/sles11.2-x86_64-install-compute/linux <node>:/boot/xcat-sles-kernel
       scp /tftpboot/xcat/osimage/sles11.2-x86_64-install-compute/initrd <node>:/boot/xcat-sles-initrd

    * Now ssh to the node and edit ``/boot/grub/grub.conf``:
        a. In the same format as the other stanzas, add a new stanza with lines: title, root, kernel and params, initrd. You should consider not making this stanza the default, so that if you have trouble with rcons, you can always boot into the default OS again to fix it.
        b. For the kernel params in the stanza you are adding, copy them from the bootparams display on the MN. Replace ``!myipfn!`` with the private IP address of the MN.
        c. To the kernel parameters, add the static IP for the node, e.g.: ``hostip=10.54.51.5 gateway=10.54.51.1 netmask=255.255.255.192``
        d. To the kernel parameters, also add which NIC the node will be installing over, and increase the wait time (in seconds) before the node tries to communicate over that NIC. (The switches have a long delay after a NIC state change.). For example: netdevice=eth0 netwait=90

    * Back on the MN, open a console for each node in a separate window (see previous section), and then boot the node and watch the progress: ::

       rsetboot <node> hd
       xdsh <node> reboot

      ``Note``: because of the slowness of the switches to respond to NICs coming up, the installation process will probably hang at one point. On the console, autoyast will ask if you want to retry. Wait about 15 seconds and then retry and the process should continue.


