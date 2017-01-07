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

At this point you can set up `ssh access <https://help.ubuntu.com/community/SSH/OpenSSH/Configuring>`_ from the host or continue working in the KVM console. I'm not going to cover it here for the purpose of brevity but I would recommend ssh for better productivity.

Install MariaDB
~~~~~~~~~~~~~~~

MySQL and MariaDB should work equally well for Nextcloud. While MySQL remains the standard for the LAMP stack on Ubuntu (CentOS prefers MariaDB), I decided to use MariaDB for reasons outlined in this article: https://seravo.fi/2015/10-reasons-to-migrate-to-mariadb-if-still-using-mysql 

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

First we need to configure MariaDB so it will work for Nextcloud. We will create a specific config file and hopefully comments make what is being done self-explanatory. To find out **why**, see:   https://docs.nextcloud.com/server/11/admin_manual/configuration_database/linux_database_configuration.html

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

.. code-block:: sql
   
   SHOW GLOBAL VARIABLES LIKE 'log_bin';
   SHOW GLOBAL VARIABLES LIKE 'tx_isolation';
   SHOW GLOBAL VARIABLES LIKE 'innodb_large_prefix';
   SHOW GLOBAL VARIABLES LIKE 'innodb_file_format';
   SHOW GLOBAL VARIABLES LIKE 'innodb_file_per_table';


Create the database and user. We will call the user ``oc_nextadmin`` in alignment with the use of the ``oc_`` prefix for all tables (note: oc stands for OwnCloud the project Nextcloud was forked from). 

Replace ``apassword`` with the password you will be using. This is required with a subsequent install step, however, you will only need to use use the application administrator password.

.. code-block:: sql

   CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
   CREATE USER oc_nextadmin@localhost IDENTIFIED BY 'apassword';
   GRANT ALL PRIVILEGES ON nextcloud . * TO oc_nextadmin@localhost;
   FLUSH PRIVILEGES;

We will revisit the database during the final installation. Next we'll setup Apache.


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

With the following content as recommended in the `Nextcloud installation manual <https://docs.nextcloud.com/server/11/admin_manual/installation/source_installation.html#apache-web-server-configuration>`_:

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

Navigate to `cloud1.example.vm/php.info` in your KVM host's web browser. You should see something like: 

ADD PICTURE HERE

You don't need the file anymore so remove it.

.. code-block:: console

   sudo rm /var/www/html/test.php


Download & Install Nextcloud 11
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I'm downloading Nextcloud 11.0.0. You should their `download page <https://nextcloud.com/install/#instructions-server>`_ for a later stable version and consider that if available.

Make a downloads directory and move into it: 

.. code-block:: console

   mkdir downloads; cd downloads;

We will download the archive as well as the certificates and checksum. Replace "11.0.0" with a later version as desired.

.. code-block:: console

   wget https://nextcloud.com/nextcloud.asc
   wget https://download.nextcloud.com/server/releases/nextcloud-11.0.0.tar.bz2.asc
   wget https://download.nextcloud.com/server/releases/nextcloud-11.0.0.tar.bz2.sha256
   wget https://download.nextcloud.com/server/releases/nextcloud-11.0.0.tar.bz2

Verify checksum integrity:

.. code-block:: console

   sha256sum -c nextcloud-11.0.0.tar.bz2.sha256 < nextcloud-11.0.0.tar.bz2

You should see:

.. code-block:: console
   
   nextcloud-11.0.0.tar.bz2: OK

Confirm PGP signature:

.. code-block:: console

   gpg --import nextcloud.asc
   gpg --verify nextcloud-11.0.0.tar.bz2.asc nextcloud-11.0.0.tar.bz2

You should see a response including these lines:

.. code-block:: console
   
   gpg: Good signature from "Nextcloud Security <security@nextcloud.com>"
   gpg: WARNING: This key is not certified with a trusted signature!
   gpg:          There is no indication that the signature belongs to the owner.

Expand the archive, changing the output destination to the web server directory. Note the ``v`` flag is for verbose and is optional. 

.. code-block:: console

   sudo tar -xvjf nextcloud-11.0.0.tar.bz2 -C /var/www/

Temporarily change the owner of the Nextcloud directory to the HTTP user then move there to run the command line installation.

.. code-block:: console

   sudo chown -R www-data:www-data /var/www/nextcloud/

.. code-block:: console

   cd /var/www/nextcloud/

Run the command line installation as the HTTP user, changing the capitalized passwords to your own. Note again that you will need to use the ``admin-pass`` regularly but not the ``database-pass``.

.. code-block:: console

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

Finally, add the host name and static IP to the by editing the config file:

.. code-block:: console

   sudoedit /var/www/nextcloud/config/config.php

Update the ``trusted_domains`` variable as follows:

.. code-block:: php

   'trusted_domains' =>
   array (
     0 => 'localhost',
     1 => '192.168.122.20',
     2 => 'cloud1.example.vm'
   )

Let apache reload its configurations.

.. code-block:: console

   sudo service apache2 reload

Install Confirmation & Login
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

From your KVM host's web browser navigate to https://cloud1.example.vn/nextcloud

Since your SSL certificate is not signed by a certificate authority your browser should tell you the connection is not secure. 

ADD PICTURE

Add the exception and continue. 

You should see a login screen where you should enter the app admin info and click "Log in".

If you see this final picture you've succeeded! Now you can go ahead and try it out - add some users and play around with files. You'll want to start syncing with a `client <https://nextcloud.com/install/#install-clients>`_ to really test it. In future articles I plan to write more on getting into production. 

