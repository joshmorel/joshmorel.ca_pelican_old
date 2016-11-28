Style my GitHub Page with Pelican-Themes
########################################

:date: 2016-11-28 07:34
:modified: 2016-11-28 07:34
:tags: python, pelican, github, disqus
:category: I-want-to-
:authors: Josh Morel
:slug: style-my-github-page
:summary: Step by step instructions for adding style to your GitHub Page with pelican themes


Background
----------

In my `first article <{filename}/create-github-page.rst>`_ I detailed how to create your GitHub Personal Page with the static site generation Python package named `Pelican <http://docs.getpelican.com>`_ . 

Here, I'll show you how to apply one of the many themes available to make your page beautiful. This is equally applicable to styling a project page.

Installing
----------

Before doing anything I would recommend browsing the `themes demo site <http://www.pelicanthemes.com/>`_ first. If you find the one you want that makes everything simpler before doing any local work.

To install, the best bet is to clone the entire repo and all the submodules (with *--recursive* option):

.. code-block:: console
   
   git clone --recursive https://github.com/getpelican/pelican-themes.git

The `README <https://github.com/getpelican/pelican-themes>`_ recommends adding the absolute path to the theme in your *pelicanconf.py* file.

This will work but for me since I'm working in both Linux and Windows environments I'd rather use the `pelican-themes <http://docs.getpelican.com/en/stable/pelican-themes.html>`_ utility.

So for pelican-bootstrap3:

.. code-block:: console
   
   pelican-themes -i /path/to/pelican-themes/pelican-bootstrap3

You can confirm installation by:

.. code-block:: console

   $ pelican-themes -l
   notmyidea
   pelican-bootstrap3
   simple
   
Applying the Theme
------------------

Adding the theme is as simple as editing the **publishconf.py** file with the following line:

.. code-block:: python
   
   THEME = 'pelican-bootstrap3'

Of course, replace ``pelican-bootstrap3`` with whatever theme you want to try out.

Now build and serve your styled site as described in my `first article <{filename}/create-github-page.rst>`_.

BONUS - Comments with Disqus
----------------------------

You can bring some non-static commenting functionality to your static site with `Disqus <https://disqus.com/>`_. With pelican-bootstrap3, as well as the pre-installed theme **notmyidea**, it is very simple to add. For other themes, you'll have to double-check.

First create your `Disqus <https://disqus.com/>`_ account and site, taking note of your site's **shortname**.

Now, in your **pelicanconf.py** add the line:

.. code-block:: python
   
   DISQUS_SITENAME = 'shortname'

Where ``shortname`` is your Disqus site's shortname.

When you build & serve your site locally, Pelican will include the HTML & JavaScript necessary to use Disqus in every article. Open http://localhost:8000 and navigate to any article. At the bottom you should see something like:

   **Comments**

   We were unable to load Disqus. If you are a moderator please see our troubleshooting guide.

That's okay because we're using relative paths when building locally.

Let's build with intent to publish using:

.. code-block:: console

   cd /path/to/ghpages
   pelican content/ -s publishconf.py

Now, open `http://localhost:8000/my-post.html` where `my-post.html` is one of your posts. If you set everything up alright on Disqus and provided the right shortname you should see Disqus ready to take comments.

Next Steps
----------

I think my next Pelican-related post will be about customizing your site with your own CSS and/or JS.
