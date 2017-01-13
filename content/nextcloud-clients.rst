Install Client for Nextcloud on Ubuntu Desktop
##############################################
:date: 2017-01-07 10:45
:modified: 2017-01-07 18:34
:tags: nextcloud, linux, ubuntu
:category: Private cloud 
:slug: install-client-nextcloud-ubuntu-desktop
:authors: Josh Morel
:summary: Step-by-step instructions for installing a desktop client for Nextcloud on Ubuntu

Background
----------

In the `first article in this series <{filename}/install-ubuntu-desktop-client-for-nextcloud.rst>`_ I described how to to install Nextcloud on Ubuntu 16.04 server. To actuall use Nextcloud you need one (or more) clients.

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


Client Usage
------------


The client can be opened by Open the client
The client is set to run on startup so you can reboot.