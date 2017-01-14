Install and Setup Nextcloud Clients
###################################
:date: 2017-01-07 10:45
:modified: 2017-01-07 18:34
:tags: nextcloud, linux, ubuntu
:category: Private cloud 
:slug: install-client-nextcloud-ubuntu-desktop
:authors: Josh Morel
:summary: Step-by-step instructions for installing a desktop client for Nextcloud on Ubuntu

Background
----------

In the `first article in this series <{filename}/install-ubuntu-desktop-client-for-nextcloud.rst>`_ I described how to to install Nextcloud on Ubuntu 16.04 server. To actually use Nextcloud you need one, or more likely multiple, clients.

In this article I will cover:

1. Enabling port forwarding to utilize the Nextcloud server
2. Installing and using the Nextcloud client on Ubuntu, Windows AND Android
3. Architecting a production-ready solution with docker

Port Forwording for the Win
---------------------------

I am still in a dev scenario with Nextcloud so I will use my VM installed in the previous article. It's being hosted on my Kubuntu Desktop, so if I want to access it from Windows or Android I'll need to set-up port-forwarding.

I'm going to use `firewalld <http://www.firewalld.org/>`_ but this is, of course, possible with `iptables <https://www.netfilter.org/projects/iptables/index.html>`_ directly or with another firewall management program.

firewalld comes with CentOS by default but for Ubuntu we need to install it.

.. code-block:: console

   sudo apt install firewalld

I'm going to make a bunch of changes to the firewalls, so if you're following along you can make it permanent with the **--permanent** flag in each.

First, let's set the default zone to ``home`` and add both my real & virtual NICs:

It is VERY IMPORTANT TO NOTE that I'm making permanent changes here. But one of the huge benefits of firewalld is that by default the changes are temporary.

.. code-block:: console

   sudo firewall-cmd --set-default-zone=home
   sudo firewall-cmd --permanent --add-interface={enp4s0,virbr0}
   sudo firewall-cmd --add-interface=enp4s0 #what about multiple?



Okay, so on home network my Kubuntu IP is 192.168.0.10. I want all http & https traffic forwarded from this IP to my Nextcloud VM which is 192.168.122.30.

Note we really are only interested in 443/https BUT i like to do both because for now,.

We need to enable the required services & ports.

?? look into --runtime-to-permanent

.. code-block:: console

   sudo firewall-cmd --add-service={http,https}
   sudo firewall-cmd --add-port={80/tcp,443/tcp}


Add masquerading & port forwarding and finally reload the settings.


.. code-block:: console

   sudo firewall-cmd --add-masquerade
   sudo firewall-cmd --add-forward-port=port=443:proto=tcp:toaddr=192.168.122.30
   sudo firewall-cmd --add-forward-port=port=80:proto=tcp:toaddr=192.168.122.30


    #forward port-based traffic to other ip e.g. =port=80:proto=tcp:toaddr=192.168.122.30

Verify & Test With Windows
--------------------------

We will verify the firewall settings are correct with my Windows laptop on the same home network. This is, of course, possible with Linux or Mac, just update the corresponding static hosts file.

First, let's make sure the VM will respond.

Enter hte following in a browser:

http://192.168.122.30

If nothing has been disabled, you should see the `Apache2 Ubuntu Default Page`: https://www.linux.com/learn/apache-ubuntu-linux-beginners

But for nextcloud to work, you need to add the following line to "C:\Windows\System32\drivers\etc\hosts\" as an administrator using your favourite Windows text editor:

.. code-block:: console

   192.168.0.10 cloud1.example.vm


Now, in a browser, enter https://cloud1.example.vm/nextcloud

You should see a log-in page if all is succesful:

ADD PICTURE HERE::::


When your happy, return to the Kubuntu desktop and make the firewall pieces permanent by:

To make it permanent do this:

.. code-block:: console

   sudo firewall-cmd --runtime-to-permanent


NOW, it's up to you because obviously, for production purposes a VM is not an option (we'll get to that later).

Client Install & Usage -- Windows
---------------------------------

With the hosts file updated, everything else in Windows hould be super straight forward.

Go to https://nextcloud.com/install/#install-clients

click "Windows". This will download the executable. When the download is done double-click to begin the install.

Finish the install.

NEED TO REMOVE SETTINGS !!!!



1. Change the default

In my last article I gave my Nextcloud server an IP of 192.168.122.30

Client Install & Usage -- Ubuntu
--------------------------------




Installing the client on my Windows Laptop and Android Phone was super straight-forward with instructions available from Nextcloud's installs page: https://nextcloud.com/install/#install-clients.

Installation the client on Ubuntu was not straight forward, unfortunately. At the time of this writing the instructions involve cloning the Nextcloud client repo which has the ownCloud client repo as a submodule. The install involves building the ownCloud client from source while applying the Nextcloud theming.

After figuring out all the dependencies and troubleshooting a few items I was able to get the ownCloud client installed and working, but... without the theming.

This process was ugly so for writing this article I decided to start from scratch. I will install the ownCloud client from the a package repo then demonstrate connectivity. Finally, I will provide optional steps for theming.

Note: This article may quickly become obsolete once a Nextcloud client is made available through the Ubuntu repos.


Install ownCloud Client
-----------------------

`ownCloud <https://owncloud.org/>`_ is the project from which Nextcloud was forked. For more on that whole story check out `this article <https://serenity-networks.com/goodbye-owncloud-hello-nextcloud-the-aftermath-of-disrupting-open-source-cloud-storage/>`_.

Info on how to install for different distributions is available on the openSuse website: https://software.opensuse.org/download/package?project=isv:ownCloud:desktop&package=owncloud-client

This will cover the necessary steps for *Ubuntu 16.04* but the instructions are provided at your own risk so I would visiting the web page to preview what you will be doing none-the-less.

Add the package resource list from the repository

.. code-block:: console

   sudo sh -c "echo 'deb http://download.opensuse.org/repositories/isv:/ownCloud:/desktop/Ubuntu_16.04/ /' > /etc/apt/sources.list.d/owncloud-client.list"


Update your repository lists:

.. code-block:: console

   sudo apt update

You will get an error indicating that the signature couldn't be verified. I'm okay with this as I only plan on installing this one package from the repo. You can add the key for the repo but there are some risks so make that decision on your own.

.. code-block:: console

   W: GPG error: http://download.opensuse.org/repositories/isv:/ownCloud:/desktop/Ubuntu_16.04  Release: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 4ABE1AC7557BEFF9
   E: The repository 'http://download.opensuse.org/repositories/isv:/ownCloud:/desktop/Ubuntu_16.04  Release' is not signed.
   N: Updating from such a repository can't be done securely, and is therefore disabled by default.
   N: See apt-secure(8) manpage for repository creation and user configuration details.

Finally, install from the repo:

   sudo apt install owncloud-client


Client Install & Usage -- Android
---------------------------------

Let's also do it in Android!!!!


