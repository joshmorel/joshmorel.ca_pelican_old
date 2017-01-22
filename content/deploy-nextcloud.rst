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

Login to your instance and create a non-root user with sudo privileges. For the purposes of this tutorial we'll pretend this user is called ``mrcloud`` and your public IP is ``111.222.111.222``.

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

ufw should be installed with Ubuntu 16.04 but disabled by default. Let's enable & set some rules.

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

For completeness I'll repeat all necessary steps, only providing explanations where my steps deviate from the first article.

Install MariaDB:

.. code-block:: console

    sudo apt install -y mariadb-server mariadb-client

``sudoedit /etc/mysql/conf.d/nextcloud.cnf`` with the following content:

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

Restart & login:

.. code-block:: console

    sudo systemctl restart mysql
    sudo mysql -uroot

Create database & admin with an alternate username (if desired) and your own password (highly recommended):


.. code-block:: mysql

    CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
    CREATE USER oc_nextadmin@localhost IDENTIFIED BY 'apassword';
    GRANT ALL PRIVILEGES ON nextcloud . * TO oc_nextadmin@localhost;
    FLUSH PRIVILEGES;


Install Apache:

    sudo apt install apache2

I'll deviate a little bit from the previous instructions as we'll be setting up a higher level of security as recommended here ADD LINK. We want to direct all HTTP traffic to HTTPS and also set up strict headers.



``sudoedit /etc/apache2/sites-available/nextcloud.conf