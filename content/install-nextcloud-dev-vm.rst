Trial Nextcloud VM  
##################
:date: 2016-12-20 19:38
:modified: 2016-12-20 19:38
:tags: nextcloud, linux
:category: Private cloud 
:slug: install-nextcloud-dev-vm
:authors: Josh Morel
:summary: Step by step instructions for installing NextCloud on a dev box.

Background
----------

I my `previous article <{filename}/create-householdwiki-vimwiki.rst>`_ I described my first steps towards independence from Microsoft Office 365 with Vimwiki as a household wiki & note taker. This would act as a replacement for OneNote specifically.

Vimwiki is very good, but a true replacement requires a private cloud with which I can access, control and protect my data. The thing that I believe can meet that need is `Nextcloud <https://nextcloud.com/>`_.

The first step is to install Nextcloud in a dev environment. The purpose is to gain comfort with the software while making better decisions for production use including architecture, system resources & add-on applications.

Prerequisites
-------------

At least some familiarity with installing Linux and working at the command line is assumed to continue. 

I completed the following using the Ubuntu Server 16.04 with LTS (Xenial Xerus). Download the ISO located here and follow along or try with a later LTS version: https://www.ubuntu.com/download/server

My guest desktop environment is `Kubuntu 16.04 <http://kubuntu.org/getkubuntu/>`_  with KVM installed following the instructions described here: https://help.ubuntu.com/community/KVM/Installation

You should be able to achieve the same using `Virtual Box <https://www.virtualbox.org/>`_ on Windows, Mac and most Linux distros using largely similar steps.

VM Set-up
~~~~~~~~~

Open the KVM Virtual Machine Manager GUI:

.. code-block:: console
   
   virt-manager

In the GUI Click the **Create a new virtual machine** button.

**Step 1**: Leave the default as *Local install media (ISO image or CDROM)* and continue.

**Step 2**: Select the *Use ISO image:* option and type or paste the path to the  downloaded ISO image and continue.

INSERT PICTURE HERE

**Step 3**: Leave the defaults to "1024" *MiB* and "1" *CPUs*.

INSERT PICTURE HERE

This will definitely be important when considering where you will host your production environment. If with a cloud provider (as opposed to at home with a VPN), doubling the RAM will double your monthly cost, so you will want to play around with this and see what works for your needs.

INSERT PICTURE HERE

**Step 4**: Use default *Create a disk image for the virtual machine*. For initial development purposes I will only use 20.0 GiB. 

INSERT PICTURE HERE

**Step 5**: Give it a meaningful name, I'll use "cloud1", and click finish.

INSERT PICTURE HERE

KVM should open a console immediately to begin the install.

Installing the OS
~~~~~~~~~~~~~~~~~

Instead of going through every Ubuntu install screen here I will point you in the direction of this tutorial on howtoforge: https://www.howtoforge.com/tutorial/ubuntu-16.04-xenial-xerus-minimal-server/

The only major difference is that we will be using "cloud1.example.vm" for the *hostname*.
 
Also, you can skip **8. Configure the Network** as I'll be covering that part explicitly in the next section.

Note that we could install the entire LAMP (Linux, Apache, MySQL, PHP) stack during the Ubuntu install **Software selection** step. This would be a simpler experience but I'll be doing each install & configuration deliberately for control, understanding and also because I want MariaDB (I'll describe why later).

Network Setup
~~~~~~~~~~~~~

Before continuing with the network setup, we should upgrade the packages installed from the distribution's ISO: 

.. code-block:: console

   sudo apt update
   sudo apt upgrade

KVM will give us a dynamic IP but we need a static one for our server. 

The KVM NAT network is `192.168.122.0` and the guest's interface name is `ens3`. If these are  different for you please make the appropriate substitutions.

Edit the network interfaces file:

.. code-block:: console

   sudoedit /etc/network/interfaces

Update the description following the commented line `# The primary network interface`: 
 
.. code-block:: console

   auto ens3
   iface ens3 inet static
           address 192.168.122.20
           netmask 255.255.255.0
           network 192.168.122.0
           broadcast 192.168.122.255
           gateway 192.168.122.1
           dns-nameservers 8.8.8.8 8.8.4.4


Restart the networking service:

.. code-block:: console

   sudo service networking restart

Next we want to add hostnames but first let's test that the networking is still working.

From the guest:

.. code-block:: console

   ping www.google.com

From the host:

.. code-block:: console

   ping 192.168.122.20

In production we will rely on DNS, but for initial development we will add an entry in the `hosts` file of the our KVM **host** for static hostname look-up:

.. code-block:: console

   sudoedit /etc/hosts

Add this entry:

.. code-block:: console

   192.168.122.20 cloud1.example.vm cloud1

Test that this works from the KVM host with:

.. code-block:: console

   ping cloud1.example.vm

You should get a response similar to:

.. code-block:: console

   PING cloud1.example.vm (192.168.122.20) 56(84) bytes of data.
   64 bytes from cloud1.example.vm (192.168.122.20): icmp_seq=1 ttl=64 time=0.292 ms
   64 bytes from cloud1.example.vm (192.168.122.20): icmp_seq=2 ttl=64 time=0.367 ms


Install MariaDB
~~~~~~~~~~~~~~~

MySQL and MariaDB should work equally well for Nextcloud. While MySQL remains the standard for the LAMP stack on Ubuntu (CentOS prefers MariaDB), I decided to use MariaDB for reasons outlined in this article: https://seravo.fi/2015/10-reasons-to-migrate-to-mariadb-if-still-using-mysql 

First, install the server & client packages:

.. code-block:: console
   
   sudo apt install mariadb-server mariadb-client

The service should be running, you can check using:

.. code-block:: console
   
   systemctl status mysql

Run this script to complete a secure installation:

.. code-block:: console
   
   sudo mysql_secure_installation


Answer the prompts as follows. Note that the MariaDB root user is different from the root user. However, MariaDB is now installed with the root user authenticated using the `unix_socket <https://mariadb.com/kb/en/mariadb/unix_socket-authentication-plugin/>`_ plugin. We will therefore leave the MariaDB root user password empty. After completing this script, MariaDB root can only be accessed locally by root or users who can gain root privileges.

* Enter current password for root (enter for none) **Enter** 
* Set root password? [Y/n] **n**
* Remove anonymous user? [Y/n] **Enter (default - Y)**
* Disallow root login remotely? [Y/n] **Enter**
* Disallow root login remotely? [Y/n] **Enter**
* Remove test database and access to it? [Y/n] **Enter**
* Reload privileges table now? [Y/n] **Enter**



