Deploy Nextcloud 11 to Private Cloud Provider
#############################################
:date: 2017-01-23 11:40
:modified: 2017-01-23 11:40
:tags: nextcloud
:category: Private cloud
:slug: deploy-nextcloud
:authors: Josh Morel
:summary: Step-by-step instructions for deploying Nextcloud 11 on an Ubuntu 16.04 to a private cloud provider.
:series: Nextcloud

Background
----------

So far in this series, we have covered how to `install a Nextcloud server <{filename}/install-nextcloud-dev-vm.rst>`_ and `sync files across clients on multiple devices <{filename}/nextcloud-clients.rst>`_. This was all very good for becoming comfortable in administering Nextcloud but we still don't have anything useful.

In this article we'll deploy to production. In the end we'll have a great sync-n-share application over which we have control.

Note that I did mentioned that for production usage I would be using docker as a means to containerize my application for ease of update & potential future migration to a homer server. I decided to delay that option because I was eager to get my Nextcloud instance up and running. I do certainly intend to revisit this at a later date.

I've deployed my Nextcloud an a Digital Ocean droplet. If you are hosting yours in a home network you will have different considerations but many of these steps will still apply.

Before I continue I must recognize that Digital Ocean does have a one-click deploy for `ownCloud <https://www.digitalocean.com/products/one-click-apps/owncloud/>`_. My guess is at some point they will have one for Nextcloud as well. While this option would be simpler, as mentioned in a previous article I want control & and I want understanding.

So what will I cover in this article:

1) Deploying to a private cloud
2) Getting an SSL certificate from Let's Encrypt
3) Additional considerations for security
4) Installing & using non-default modules

Prerequisites
-------------

Spin up an instance of Ubuntu 16.04 (or greater) with a private cloud provider (Digital Ocean, VPSie, vultr, etc). Results may vary with versions of Ubuntu later than 16.04. Please review the `Nextcloud document <https://docs.nextcloud.com/server/11/admin_manual/installation/php_55_installation.html>`_ for CentOS-specific requirements if that is your preference.

Login to your instance and create a non-root user with sudo privileges. For the purposes of this tutorial we'll pretend this user is called "mrcloud" and your public IP is "111.222.111.222".

Obtain or use your own domain name, adding a CNAME record entry for "cloud.yourdomain.tld".

Because you'll be using a public IP I would also:

* disable remote root login & password-based authentication with ssh
* set-up iptable rules to filter unexpected traffic

After logging in as root & changing the root password:

.. code-block:: console

    adduser mrcloud
    usermod -aG sudo mrcloud

Let's copy our ssh public key from our desktop:

.. code-block:: console

    ssh-copy-id mrcloud@111.222.111.222

Enter the password created earlier when prompted & login.

.. code-block:: console

    ssh mrcloud@111.222.111.222

``sudoedit /etc/ssh/sshd_config`` to include the following lines :

.. code-block:: console

    PermitRootLogin no
    PasswordAuthentication no

Reload ssh and exit:

.. code-block:: console

    sudo systemctl reload ssh
    exit

Back on your desktop, add to the ``~/.ssh/config`` file:

.. code-block:: console

    Host nextcloud
        HostName 111.222.111.222
        User mrcloud
        Port 22

Now you can log-in with:

.. code-block:: console

    ssh nextcloud

For more ssh usage options check out `article from Digital Ocean <https://www.digitalocean.com/community/tutorials/ssh-essentials-working-with-ssh-servers-clients-and-keys>`_.

We also want to implement a basic firewall to allow only incoming http, https & ssh. We'll use the aptly-named uncomplicated firewall - `ufw <https://help.ubuntu.com/community/UFW>`_ - on Ubuntu. On CentOS you'll want to look into `firewalld <http://www.firewalld.org/>`_.

ufw should be installed with Ubuntu 16.04 but disabled by default. Let's enable & set some rules to allow only the incoming traffic we expect:

.. code-block:: console

    sudo ufw enable
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow ssh
    sudo ufw allow http
    sudo ufw allow https

If you want to be even more secure you can restrict based on incoming IP. For example let's say your home's public IP is ``99.88.77.66`` and that is the only place you expect ssh access to originate from.

.. code-block:: console

    sudo ufw delete allow ssh
    sudo ufw allow from 99.88.77.66 to any port 22 proto tcp



Complete Nextcloud Installation
-------------------------------

Installing the Nextcloud server was covered in-depth in my `first article in the series <{filename}/install-nextcloud-dev-vm.rst>`_.

For completeness I'll repeat all necessary steps which I outlined in detail in a super compact form (including use of here documents & sed for editing config files).  I will only provide explanations where my steps from the first article.

Install **MariaDB**:

.. code-block:: console

    sudo apt install -y mariadb-server mariadb-client

Store the necessary config options for Nextcloud operation:

.. code-block:: console

    cat << EOF | sudo tee /etc/mysql/conf.d/nextcloud.cnf
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
    EOF

Restart the login as root:

.. code-block:: console

    sudo systemctl restart mysql

.. code-block:: console

    sudo mysql -uroot

Create the database and user, replacing the username (optional) and password (highly recommended) with your own then exit.

.. code-block:: mysql

    sudo mysql -uroot

    CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
    CREATE USER oc_nextadmin@localhost IDENTIFIED BY 'DBPASS';
    GRANT ALL PRIVILEGES ON nextcloud . * TO oc_nextadmin@localhost;
    FLUSH PRIVILEGES;
    exit

Install **Apache**:

.. code-block:: console

    sudo apt -y install apache2


Create the Nextcloud virtual host configuration file. Note that I've upgraded this from the previous article as per the `SSL-specific recommendations from Nextcloud <https://docs.nextcloud.com/server/11/admin_manual/configuration_server/harden_server.html#use-https>`_. As we're public, we now definitely want our communication to be transmitted via SSL by redirecting HTTP traffic. We also will add the `HTTP Strict Transport Security header <https://en.wikipedia.org/wiki/HTTP_Strict_Transport_Security>`_.

.. code-block:: console

    sudoedit /etc/apache2/sites-available/nextcloud.conf

Add the following, replacing "yourdomain.tld" with your actual domain.

.. code-block:: aconf

    Alias /nextcloud "/var/www/nextcloud/"

    # Redirecting all HTTP traffic to HTTPS
    <VirtualHost *:80>
            ServerName "cloud.yourdomain.tld"
            Redirect permanent "/" "https://cloud.yourdomain.tld/"
    </VirtualHost>

    <VirtualHost *:443>
            ServerName "cloud.yourdomain.tld"

            SSLEngine on

            # HSTS (mod_headers is required) (15768000 seconds = 6 months)
            Header always set Strict-Transport-Security "max-age=15768000"

            ErrorLog ${APACHE_LOG_DIR}/error.log
            CustomLog ${APACHE_LOG_DIR}/access.log combined

            <Directory /var/www/nextcloud/>
              Options +FollowSymlinks
              AllowOverride All
              <IfModule mod_dav.c>
                Dav off
              </IfModule>

            SetEnv HOME /var/www/nextcloud
            SetEnv HTTP_HOME /var/www/nextcloud
            </Directory>
    </VirtualHost>


Enable the site, required modules & restart apache. Note that last time we also denabled the default-ssl site. But because we have defined ssl usage in the ``.conf`` file this is no longer necessary. There are some optional steps we can take towards further improving apache security which I will detail at the end.

.. code-block:: console

    sudo a2ensite nextcloud.conf
    sudo a2enmod rewrite headers env dir mime ssl
    sudo service apache2 restart


Install all required **PHP 7.0** modules:

.. code-block:: console

    sudo apt -y install php7.0-common php7.0-cli php7.0-bz2 php7.0-curl php7.0-gd php7.0-intl php7.0-mbstring php7.0-mcrypt php7.0-mysql php7.0-mysql php7.0-xml php7.0-zip libapache2-mod-php7.0


Download & verify the bz2 archive for the latest stable version of Nextcloud server from: https://nextcloud.com/install/#instructions-server

Once you have downloaded and verified the integrity of the archive, untar it to the final location (replacing 11.X.Y with the latest version number).

.. code-block:: console

    sudo tar -xvjf nextcloud-11.X.Y.tar.bz2 -C /var/www/

Change the ownership to the apache user then move to that directory to complete the final install.

.. code-block:: console

    sudo chown -R www-data:www-data /var/www/nextcloud
    cd /var/www/nextcloud

Complete the install with ``occ``, replacing the capitalized password with your own.

.. code-block:: console

    sudo -u www-data php occ maintenance:install \
    --database "mysql" --database-name "nextcloud" \
    --database-user "oc_nextadmin" --database-pass "DBPASS" \
    --admin-user "nextadmin" --admin-pass "ADMINPASS"


Harden the security of the server by running the script that is recommended in the `Nextcloud manual <https://docs.nextcloud.com/server/11/admin_manual/installation/installation_wizard.html#strong-perms-label>`_.

Copy the entire script text (which starts ``#!/bin/bash``) to a file say ``nextcloud_harden.sh``, then make it executable & execute it:

.. code-block:: console

   chmod +x nextcloud_harden.sh
   sudo ./nextcloud_harden.sh

``sudoedit /var/www/nextcloud/config/config.php`` to add the public IP and name to the ``trusted_domains`` variable, making sure to use your proper IP & domain name.

.. code-block:: console

   'trusted_domains' =>
   array (
     0 => 'localhost',
     1 => '111.222.111.222',
     2 => 'cloud.yourdomain.tld',
   ),

Finally, tell Apache to reload configurations:

.. code-block:: console

    sudo service apache2 reload


Confirm the installation by visiting https://cloud.yourdomain.tld/nextcloud. As in our previous articles you'll need to add the security exception for the self-signed SSL certificate. But now that you know the install actually worked, let's get certified!

Getting Certified with Let's Encrypt
------------------------------------

`Let's Encrypt is a free, automated and open Certificate Authority <https://letsencrypt.org/>`_. Pretty awesome. We can follow the `certbot <https://certbot.eff.org/#ubuntuxenial-apache>`_ instructions to install


.. code-block:: console

    sudo apt -y install python-letsencrypt-apache

Then run the program:

.. code-block:: console

    sudo letsencrypt --apache

You should only have the one domain to select. Continue, and provide your email. Then try accessing the site again. It should be obvious that the certificate has been verified by an CA due to the green lock icon in the top-left corner.

This will expire after 90 days so you will need to renew. I will leave that piece up to you. You can find some useful documentation here: https://certbot.eff.org/docs/using.html#renewal


**So you are good to go!** You can start using your production instance right away with a few `desktop <{filename}/nextcloud-clients.rst>`_ or `mobile <https://nextcloud.com/install/#install-clients>`_ clients.

I have experienced excellent usability & performance so far with only 512MiB of RAM and 1 CPU on a 20GiB SSD Digital Ocean droplet. Of course it's just me and I only have about 2GiB of files in play so results will certainly vary.

Additional Security Considerations
----------------------------------

I would recommend reviewing Nextcloud's `Hardening and Security Guidance <https://docs.nextcloud.com/server/11/admin_manual/configuration_server/harden_server.html>`_ and decide what else you may want to apply.

Some additional steps not explicitly covered that apply to Apache more generally include:


Disaster Recovery
-----------------

You'll definitely want to consider disaster recovery. I have yet to put that in place but certainly plan to. Recommendations are provided in the Nextcloud administration manual: https://docs.nextcloud.com/server/11/admin_manual/maintenance/index.html



