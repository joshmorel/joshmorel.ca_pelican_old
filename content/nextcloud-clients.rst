Install and Use Nextcloud Desktop Clients
#########################################
:date: 2017-01-07 10:45
:modified: 2017-01-07 18:34
:tags: nextcloud, linux, ubuntu
:category: Private cloud 
:slug: install-nextcloud-clients
:authors: Josh Morel
:summary: Step-by-step instructions for installing & using Nextcloud clients

Background
----------

In the `first article in this series <{filename}/install-nextcloud-dev-vm.rst>`_ I described how to to install Nextcloud on Ubuntu 16.04. To actually make use of Nextcloud's sync-n-share capabilities you need a few clients.

In this article I will cover:

1. Enabling port forwarding to utilize the dev Nextcloud server I've installed on a VM
2. Installing and using the Nextcloud clients on Ubuntu and Windows
3. Architecting a production-ready solution with docker

Port Forwarding with firewalld
------------------------------

To make requests to the Nextcloud server on my VM from devices other than the host, I need to set up port-forwarding on my Kubuntu Desktop host.

I'm going to use `firewalld <http://www.firewalld.org/>`_ but you can use `iptables <https://www.netfilter.org/projects/iptables/index.html>`_ and/or `ufw <https://help.ubuntu.com/community/UFW>`_ commands or some other firewall management program.

firewalld comes with CentOS by default but for Ubuntu we need to install it.

.. code-block:: console

   sudo apt install firewalld

An excellent introduction to ``firewalld`` is available through this `Digital Ocean article <https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-using-firewalld-on-centos-7>`_

My objective is to forward http/https requests sent to my Kubuntu Desktop (192.168.0.10) to the Nextcloud VM (192.168.122.30). If this were a long term thing we'd want to make my Desktop IP static, but because this is just for trial purposes I'll skip that step.

First, let's set the default zone from ``public`` to ``home``.

.. code-block:: console

   sudo firewall-cmd --set-default-zone=home
   sudo firewall-cmd --permanent --add-interface={enp4s0,virbr0}
   sudo firewall-cmd --add-interface=enp4s0 #what about multiple?

Let's add the required services and port/protocols.

.. code-block:: console

   sudo firewall-cmd --add-service={http,https}
   sudo firewall-cmd --add-port={80/tcp,443/tcp}

We will enable masquerading to allow our system to forward packages, then add the specific port-forwarding rules for http/https.

.. code-block:: console

   sudo firewall-cmd --add-masquerade
   sudo firewall-cmd --add-forward-port=port={80,443}:proto=tcp:toaddr=192.168.122.30

By running ``firewall-cmd --list-all`` you should see something like:

.. code-block:: console

   home (default, active)
     interfaces: enp4s0 virbr0
     sources:
     services: dhcpv6-client http https mdns samba-client ssh
     ports: 80/tcp 443/tcp
     protocols:
     masquerade: yes
     forward-ports: port=80:proto=tcp:toport=:toaddr=192.168.122.30
           port=443:proto=tcp:toport=:toaddr=192.168.122.30
     icmp-blocks:
     rich rules:


Verify Port-forwarding with Windows
-----------------------------------

I'm verifying the firewall settings are correct with my Windows laptop on the same home network. This is, of course, possible with Linux or Mac, just substitute the step for updating the static hosts file.

First, let's make sure port-forwarding is working.

Enter this url in any web browser: http://192.168.122.30

If nothing has been disabled, you should see the `Apache2 Ubuntu Default Page <https://www.linux.com/learn/apache-ubuntu-linux-beginners>`_

But for Nextcloud to work you need use the hostname because 192.168.0.10 is not one of Nextcloud's trusted domains.

In Windows, we do this by appending the following line to "C:\Windows\System32\drivers\etc\hosts":

.. code-block:: console

   192.168.0.10 cloud1.example.vm

Now, in a browser, enter https://cloud1.example.vm/nextcloud

You will need to add the security certificate exception as we did in the `previous article <{filename}/install-nextcloud-dev-vm.rst>`_

If successful, you should see a login page. Leave this open as we'll be using it later.

Back on the Ubuntu VM host, if you want, you can make the firewall changes permanent. If not the settings will be be reset at the next system reboot.

.. code-block:: console

   sudo firewall-cmd --runtime-to-permanent

Create a Non-Admin User
-----------------------

If you haven't yet. You should create a non-admin Nextcloud user. Log in as ``nextadmin``.

Click on "nextadmin" in the top-right corner and select "Users". The first line on the "Users" page allows you to create a new user very easily:

.. image:: {filename}/images/nextcloud_create_user.png
   :alt: image: Nextcloud create user

----

Let's create a user called "cloudboy" and give him a password. You can also create groups but we won't bother with that now.

Try logging out then back in as "cloudboy" to confirm it worked.


Client Install & Usage -- Windows
---------------------------------

We already updated the host file, so everything else in Windows will be super straight forward.

Go to https://nextcloud.com/install/#install-clients

click "Windows". This will download the executable. When the download is done double-click to begin the install and complete the install with all the default options selected. A succesful install should end with the launch of a "Nextcloud Connection Wizard":

.. image:: {filename}/images/nextcloud_wizard_address.png
   :alt: image: Nextcloud connection wizard address

----

Enter the URL: https://cloud1.example.vm/nextcloud

You will need to accept the untrusted certificate then enter cloudboy's username and password.

The installer will ask you what to sync. You can keep or change the defaults. Once this is done, the files should be downloaded in the local folder.


.. image:: {filename}/images/nextcloud_wizard_sync.png
   :alt: image: Nextcloud connection wizard sync options

----

Try adding files through both the web interface and local filesystem. It should all be very intuitive.


Client Install & Usage -- Ubuntu
--------------------------------

At the time of this writing, there is no installable Nextcloud binary for Ubuntu. I was able to install from source after some mucking about with dependencies and other troubleshooting but I wouldn't recommend it.

Let's instead use the ownCloud client available through the Ubuntu package repository which works just fine with the Nextcloud server (for now). As the projects diverge this may change, but hopefully at that point there we can easily install a Nextcloud client on Ubuntu.

If you want some background on the `ownCloud <https://owncloud.org/>`_ and Nextcloud split check out `this article <https://serenity-networks.com/goodbye-owncloud-hello-nextcloud-the-aftermath-of-disrupting-open-source-cloud-storage/>`_.

.. code-block:: console

   sudo apt install owncloud-client

On next reboot, the client will run automatically. Until then, you can run ``owncloud`` from the console or find the client in the start menu:

.. image:: {filename}/images/owncloud_start.png
   :alt: image: Owncloud in start menu

----

You will be presented with the "ownCloud Connection Wizard". Not surprisingly, the steps are much the same as the Nextcloud Windows client:

.. image:: {filename}/images/owncloud_wizard_address.png
   :alt: image: Nextcloud connection wizard address

----

1. Enter the URL - https://cloud1.example.vm/nextcloud
2. Accept the untrusted certificate
3. Enter cloudboy's username and password
4. Accept or modify the syncing defaults

Try changing or adding files on one device and you **should** see the file downloaded on the other device very quickly!

Architecture for Production - Docker
------------------------------------

