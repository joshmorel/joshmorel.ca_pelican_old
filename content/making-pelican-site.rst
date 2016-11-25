Make a GitHub Personal Pages with Python
########################################

:date: 2016-11-25 07:41
:modified: 2016-11-25 7:41
:tags: python, pelican
:category: I-want-to
:authors: Josh Morel
:summary: Step by step instructions for creating a personal site hosted on github pages


Background
----------

Pelican is a Python package which helps you generate static sites. `GitHub Pages <https://pages.github.com/>`_ will host that static site for you for free. These two combined allow someone to create a beautiful site without relying on something like Wordpress or needing to know HTML, CSS or JavaScript.   

Prerequisites include:

1. `Python <https://www.python.org>`_ (2.7 works for now)
2. `Git <https://git-scm.com/>`_
3. A `GitHub <https://github.com/>`_ account 
4. Pelican

I'll leave the first three to you. 

Pelican
-------

You can install Pelican with pip or your package manager on some Linux distribution. 

On Ubuntu try:

.. code-block:: bash
   
   sudo apt-get install python-pelican

With pip:

.. code-block:: bash
   
   pip install pelican

Restart your shell so you can use Pelican at the command line. There's a lot more info on their `docs site <http://docs.getpelican.com>`_ so if you have challenges head there.

GitHub Pages init
---------------------------

To create a GitHub pages site you need to at least create a new repository named *you.github.io* where *you* is substituted by your GitHub user name. In this implementation we will actually create two. 

1. **you.github.io-src** will contain the Pelican project used to generate the static website
2. **you.github.io** will contain the website itself output from a Pelican project

`Create <https://github.com/new>`_ both of these then return to the terminal.

Clone the source repo into a ghpages directory.

.. code-block:: bash
   
   git clone https://www.github.com/you/you.github.io-src ghpages; cd ghpages
   
Now you can add the website repository as a `git submodule <https://git-scm.com/book/en/v2/Git-Tools-Submodules>`_ in the project's output directory while will make your future workflow quite nice.

.. code-block:: bash
   
   git submodule add https://www.github.com/you/you.github.io-src output

Starting the Pelican Site Project
---------------------------------

Make sure you're in your source repo and generate the using the pelican quickstart utility.

There will be several prompts, the following answers are important to mimic:

* Where do you want to create your new web site? **[.]**
* What will be the title of this website **http://you.github.io**
* Do you want to generate a Fabfile/Makefile to automate generation and publishing? **Y**
* Do you want to auto-relate & simpleHTTP script to assist with theme and site development? **Y**
* Do you want to upload your website using ...? **Y for only GitHub Pages**
* Is this your personal page (username.github.io)? **Y**
