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

I my last article I wrote on steps to take to move away from dependence on Windows Office 365 with Vimwiki in place of OneNote.

I've really been liking it so far, but I need to go further. The thing that sparked my interest is `Nextcloud <https://nextcloud.com/>`_ It is a means to have full control over all my data and make it accessible from multiple devices and also safe.

The first step is to install a dev environment for trying all this out.

Prerequisites
-------------

At least some familiarity with installing Linux and working at the command line is assumed. 

I completed the following using the Ubuntu Server 16.04 (Xenial Xerus) with LTS (Long Term Support). Download the ISO located here and follow along or try with a later LTS version: https://www.ubuntu.com/download/server

My guest desktop environment is `Kubuntu 16.04 <http://kubuntu.org/getkubuntu/>`_  with KVM installed following the instructions described here: https://help.ubuntu.com/community/KVM/Installation

You should be able to achieve the same using `Virtual Box <https://www.virtualbox.org/>`_ on Windows, Mac or  practically any Linux distro using largely similar steps.

VM Set-up
~~~~~~~~~

Open the KVM GUI Virtual Machine Manager:

.. code-block:: console
   
   virt-manager

In the GUI Click the **Create a new virtual machine** button.

**Step 1**: Leave the default as *Local install media (ISO image or CDROM)* and continue.

**Step 2**: Select the *Use ISO image:* option and type or paste the path to the  downloaded ISO image and continue.

INSERT PICTURE HERE

**Step 3**: Change the RAM to 512 MiB and continue. Note that I am trying 512 because of the option to run this on `Digital Ocean <https://www.digitalocean.com/>`_ for only $5/mo. I have no idea if this will be sufficient yet but that's why I am in just creating for development purposes. With very basic VM usage I've yet to have any issues at 512 MB.

This will definitely make installing software a little slower so feel free to use something higher if desired.

INSERT PICTURE HERE

**Step 4**: Use default *Create a disk image for the virtual machine*. For initial development purposes I will only use 10.0 GiB. 

**Step 5**: Give it a meaningful name, I'll use nextcloud2, and click finish.

KVM should open a console immediately to begin the install.

Installing the OS
~~~~~~~~~~~~~~~~~

Instead of going through every Ubuntu install screen here I will point you in the direction of this tutorial on howtoforge: https://www.howtoforge.com/tutorial/ubuntu-16.04-xenial-xerus-minimal-server/

There will be three key differences:

1) Use the appropriate language, location & keyboard layout

2) For hostname use nextcloud2.example.vm
 
3) Skip "8. Configure the Network" I'll do something similar a little later

Note that we could install the entire LAMP (Linux, Apache, MySQL, PHP) stack during the Ubuntu install **Software selection** step. This would be a simpler experience but I'll be doing each install & configuration deliberately for both control & understanding.


Apache
~~~~~~

Install Apache as well as W3M for terminal web-browsing. 

.. code-block:: console
   
   sudo apt install apache2 apache2-utils w3m

You should see the 

