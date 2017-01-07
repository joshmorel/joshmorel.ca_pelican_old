Install Nextcloud 11 on an Ubuntu VM
####################################
:date: 2017-01-07 10:45
:modified: 2017-01-07 18:34
:tags: nextcloud, linux, ubuntu, mariadb, apache, php
:category: Private cloud 
:slug: install-nextcloud-dev-vm
:authors: Josh Morel
:summary: Step-by-step instructions for installing Nextcloud 11 on an Ubuntu 16.04 VM with Apache, MariaDB & PHP 7.0.

Background
----------

In my `previous article <{filename}/create-householdwiki-vimwiki.rst>`_ I described my first steps towards separation from Microsoft Office 365. Specifically, I outlined how to set-up Vimwiki for use as a OneNote replacement.

Vimwiki is very good, but a complete replacement requires a private cloud combined with a file sync-n-share application which can allow me to control and protect my data. The sync-n-share application I want to use is `Nextcloud <https://nextcloud.com/>`_. For a well written background on Nextcloud's origin check out `this article <https://serenity-networks.com/goodbye-owncloud-hello-nextcloud-the-aftermath-of-disrupting-open-source-cloud-storage/>`_.

My first major activity is to install and start using Nextcloud in a dev VM environment so I can gain some comfort with administering and using the application.

This will be a quite a manual install compared to what is possible using the `snap package <https://www.linuxbabe.com/cloud-storage/install-nextcloud-server-ubuntu-16-04-via-snap>`_. However, I specifically want to take a deliberate approach for both my understanding and to facilitate customization.

Prerequisites
-------------

At least some familiarity with installing Linux and working at the command line is assumed for following this tutorial.

I completed the following using the Ubuntu Server 16.04 with LTS (Xenial Xerus). Download the ISO located on the Ubuntu site or try with a later LTS version: https://www.ubuntu.com/download/server

My guest desktop environment is `Kubuntu 16.04 <http://kubuntu.org/getkubuntu/>`_  with KVM installed as in the `these instructions <https://help.ubuntu.com/community/KVM/Installation>`_.

You should be able to achieve the same using `Virtual Box <https://www.virtualbox.org/>`_ on Windows, Mac and most Linux distros following roughly similar steps.

VM Set-up
~~~~~~~~~

Open the KVM Virtual Machine Manager GUI:

.. code-block:: console
   
   virt-manager

In the GUI Click the **Create a new virtual machine** button.

.. image:: {filename}/images/kvm_create.png
   :alt: image: KVM Create New

------

Creating the VM is a fairly straight-forward five step process:

1. Leave the default as *Local install media (ISO image or CDROM)* and continue.

2. Select the *Use ISO image:* option and type or paste the path to the downloaded ISO image and continue.

3. Leave the defaults to "1024" *MiB* and "1" *CPUs*. The *RAM* will become quite important later as you make decisions for production deployment as it will drive cloud provider costs but for now this should be fine.

4. Use default *Create a disk image for the virtual machine*. For initial development purposes I will only use 20.0 GiB.

5. Give it a meaningful name, I'll use "cloud1", and click finish.

KVM should open a console immediately to begin the install.

In production, choices for both storage and RAM will become quite important. But at this point we can make more arbitrary choices as get used to the application.

Installing the OS
~~~~~~~~~~~~~~~~~

Instead of going through every Ubuntu install screen here I will point you in the direction of `this tutorial on howtoforge <https://www.howtoforge.com/tutorial/ubuntu-16.04-xenial-xerus-minimal-server/>`_.

The only major difference is that we will be using "cloud1.example.vm" for the *hostname*.
 
Also, you can skip **8. Configure the Network** as I'll be covering that part explicitly in the next section.

Note that we could install the entire LAMP (Linux, Apache, MySQL, PHP) stack during the Ubuntu install **Software selection** step. This would be a simpler experience but I'll be doing each install & configuration deliberately for understanding & customization (particularly with regards to use of MariaDB).

Once the install is complete, but before continuing with the network setup, we should upgrade the packages installed from the distribution's ISO. Note that this this may take some time.

.. code-block:: console

   sudo apt update
   sudo apt upgrade


Network Setup
~~~~~~~~~~~~~

KVM will give us a dynamic IP but we need a static one for our server.

The KVM NAT network is `192.168.122.0` and the guest's interface name is `ens3`. If these are  different for you please make the appropriate substitutions.

Edit the network interfaces file:

.. code-block:: console

   sudoedit /etc/network/interfaces

Update the interface description which follows the commented line "``# The primary network interface``":
 
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

In production we will rely on DNS, but for initial development we will add an entry in the `hosts` file of the KVM **host** for static hostname look-up:

.. code-block:: console

   sudoedit /etc/hosts

Add this line:

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

At this point you can set up `ssh access <https://help.ubuntu.com/community/SSH/OpenSSH/Configuring>`_ from the host or continue working in the KVM console. I'm not going to cover it here for the purpose of brevity but I would recommend ssh for better productivity.

Install MariaDB
~~~~~~~~~~~~~~~

MySQL and MariaDB should work equally well for Nextcloud. While MySQL remains the standard for the LAMP stack on Ubuntu (CentOS prefers MariaDB), I decided to use MariaDB because it is a community-driven project with a `team that delivers quicker security updates `this article <described here <https://seravo.fi/2015/10-reasons-to-migrate-to-mariadb-if-still-using-mysql>`_.

First, install the server & client packages:

.. code-block:: console
   
   sudo apt install mariadb-server mariadb-client

The service should be running, you can check using:

.. code-block:: console
   
   systemctl status mysql

On many LAMP installation tutorials you may be recommended to run the `mysql_secure_installation <http://mariadb.com/kb/en/mariadb/mysql_secure_installation>`_ script.

This is not necessary for MariaDB on Ubuntu 16.04 as:

1) MariaDB is now installed on Ubuntu with the root user authenticated using the `unix_socket <https://mariadb.com/kb/en/mariadb/unix_socket-authentication-plugin/>`_ plugin.

2) The anonymous user is no longer created on installation

3) The root users is only included for ``Host='localhost'`` on installation

4) The ``test`` database is no longer included on installation


Set-up MariaDB for Nextcloud
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

First we need to configure MariaDB so it will work for Nextcloud. We will create a specific config file with (hopefully) self-explanatory comments as to **what** is being done. To find out **why**, see:   https://docs.nextcloud.com/server/11/admin_manual/configuration_database/linux_database_configuration.html

Create in:

.. code-block:: console
   
   sudoedit /etc/mysql/conf.d/nextcloud.cnf

Add the following:

.. code-block:: console
   
   # Nextcloud database configuration file
   [mysqld]

   # disable binary logging
   skip-log-bin

   # use transaction read committed isolation
   transaction-isolation=read-committed

   # enable emojis
   innodb_large_prefix=true
   innodb_file_format=barracuda
   innodb_file_per_table=true

Restart the service:

.. code-block:: console
   
   sudo systemctl restart mysql

Login as root:

.. code-block:: console
   
   sudo mysql -uroot

Verify variables reflect the configuration file created above:

.. code-block:: mysql
   
   SHOW GLOBAL VARIABLES LIKE 'log_bin';
   SHOW GLOBAL VARIABLES LIKE 'tx_isolation';
   SHOW GLOBAL VARIABLES LIKE 'innodb_large_prefix';
   SHOW GLOBAL VARIABLES LIKE 'innodb_file_format';
   SHOW GLOBAL VARIABLES LIKE 'innodb_file_per_table';


Create the database and user. We will call the user ``oc_nextadmin`` in alignment with the use of the ``oc_`` prefix for all tables (note: oc stands for ownCloud the project Nextcloud was forked from).

Replace ``apassword`` with the password you will be using. This is required with a subsequent install step, however, for regular use you will only need to use use the application administrator password.

.. code-block:: mysql

   CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
   CREATE USER oc_nextadmin@localhost IDENTIFIED BY 'apassword';
   GRANT ALL PRIVILEGES ON nextcloud . * TO oc_nextadmin@localhost;
   FLUSH PRIVILEGES;

You can now ``exit`` as the Nextcloud install script will handle all other database tasks.

Install & Set-up Apache
~~~~~~~~~~~~~~~~~~~~~~~

There's not much to say about the Apache install so I'll cover both install & set-up together. 

Install:

.. code-block:: console
   
   sudo apt install apache2

To confirm the service is running:

.. code-block:: console

   systemctl status apache2

Create the Nextcloud site config file

.. code-block:: console

   sudoedit /etc/apache2/sites-available/nextcloud.conf

Add these lines as recommended in the `Nextcloud installation manual <https://docs.nextcloud.com/server/11/admin_manual/installation/source_installation.html#apache-web-server-configuration>`_:

.. code-block:: aconf

   Alias /nextcloud "/var/www/nextcloud/"
   
   <Directory /var/www/nextcloud/>
     Options +FollowSymlinks
     AllowOverride All
     <IfModule mod_dav.c>
       Dav off
     </IfModule>

   SetEnv HOME /var/www/nextcloud
   SetEnv HTTP_HOME /var/www/nextcloud
   </Directory>


Enable the site:

.. code-block:: console

   sudo ln -s /etc/apache2/sites-available/nextcloud.conf /etc/apache2/sites-enabled/nextcloud.conf


The Apache module ``rewrite`` is required. Nextcloud also `recommendations <https://docs.nextcloud.com/server/11/admin_manual/installation/source_installation.html#apache-web-server-configuration>`_ ``headers``, ``env``, ``dir``, ``mime`` and ``ssl``. Let's make sure all of these modules as well as the default SSL site are enabled: 

.. code-block:: console

   sudo a2enmod rewrite headers env dir mime ssl
   sudo a2ensite default-ssl
   sudo service apache2 restart


Install PHP 7.0
~~~~~~~~~~~~~~~

There are a number of `PHP modules <https://docs.nextcloud.com/server/11/admin_manual/installation/source_installation.html#apache-web-server-configuration>`_ which Nextcloud depends on. We will install them in a single command including the modules for integration with Apache & MariaDB.

.. code-block:: console

   sudo apt install php7.0-common php7.0-cli php7.0-bz2 php7.0-curl php7.0-gd php7.0-intl php7.0-mbstring php7.0-mcrypt php7.0-mysql php7.0-mysql php7.0-xml php7.0-zip libapache2-mod-php7.0

Confirm version:

.. code-block:: console

   php --version

You can see that all the required/recommended modules are installed & enabled:

.. code-block:: console

   php -m | grep -E "bz2|ctype|curl|dom|fileinfo|gd|iconv|intl|json|libxml|mbstring|mcrypt|openssl|pdo_mysql|posix|SimpleXML|xmlwriter|zip|zlib"

Confirm PHP-Apache integration:

.. code-block:: console

   echo "<?php phpinfo(); ?>" | sudo tee /var/www/html/test.php

Navigate to `<http://cloud1.example.vm/test.php>`_ in your KVM host's web browser. You should see something like:

.. image:: {filename}/images/php_info.png
   :alt: image: PHP Info

----

You don't need the file anymore so remove it.

.. code-block:: console

   sudo rm /var/www/html/test.php


Download & Install Nextcloud 11
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I'm downloading Nextcloud 11.0.0. You should go to `the Nextcloud download site <https://nextcloud.com/install/#instructions-server>`_ and download the latest stable version. I downloaded the ``.tar.bz2`` archive although there is also a ``.zip`` archive.

Verify the integrity of the file then expand the archive to the Apache server directory.

Replace ``11.0.0`` with whatever version you downloaded. Note the ``v`` - verbose - flag is optional.

.. code-block:: console

   sudo tar -xvjf nextcloud-11.0.0.tar.bz2 -C /var/www/

Temporarily change the owner of the Nextcloud directory to the HTTP user.

.. code-block:: console

   sudo chown -R www-data:www-data /var/www/nextcloud/


Run the command line installation as the HTTP user from that directory. Of course, change the capitalized passwords to your own. Note again that you will need to use the ``admin-pass`` regularly but not the ``database-pass``.

.. code-block:: console

   cd /var/www/nextcloud/
   sudo -u www-data php occ maintenance:install \
   --database "mysql" --database-name "nextcloud" \
   --database-user "oc_nextadmin" --database-pass "DBPASS" \
   --admin-user "nextadmin" --admin-pass "ADMINPASS"

If you see this, the install is successful!

.. code-block:: console

   Nextcloud is not installed - only a limited number of commands are available
   Nextcloud was successfully installed


Final Server Configuration Pieces
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Harden the security of the server as recommended in the `Nextcloud manual <https://docs.nextcloud.com/server/11/admin_manual/installation/installation_wizard.html#strong-perms-label>`_.

The last installation step is to add the host name and static IP by editing the php config file:

.. code-block:: console

   sudoedit /var/www/nextcloud/config/config.php

Update the ``trusted_domains`` variable to:

.. code-block:: php

   'trusted_domains' =>
   array (
     0 => 'localhost',
     1 => '192.168.122.20',
     2 => 'cloud1.example.vm',
   ),


Finally, tell Apache to reload configurations:

.. code-block:: console

   sudo service apache2 reload

Install Confirmation & Login
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

From your KVM host's web browser navigate to https://cloud1.example.vm/nextcloud

Since your SSL certificate is not signed by a certificate authority your browser should tell you something like:

.. image:: {filename}/images/firefox_notsecure.png
   :alt: image: Firefox not secure

----

In Firefox, for example, click "Advanced" > "Add Exception..." > "Confirm Security Exception".

When in production, you may want to consider `Let's Encrypt <https://letsencrypt.org/>`_

You should see a login screen where you can enter your app admin info and click "Log in".

If you see this final picture you've succeeded!

.. image:: {filename}/images/nextcloud_success.png
   :alt: image: Nextcloud successful install

----

Now you can go ahead and try it out - add some users and play around with file management. You'll want to start syncing with a `client <https://nextcloud.com/install/#install-clients>`_ to really test it out.

In future articles I plan to write on Nextcloud production options.
