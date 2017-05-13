Migrate Static Site from GitHub Page to S3 and CloudFront
#########################################################

:date: 2017-02-23 09:44
:modified: 2017-02-26 09:44
:tags: pelican, s3, cloudfront
:category: Web publishing 
:slug: migrate-github-page-s3-cloudflare
:authors: Josh Morel
:summary: Step by step instructions to migrate static site from github page to s3 and cloudflare
:series: Pelican

Background
----------

Back in November 2016 I wrote my first how-to article on creating a `GitHub Personal Page <{filename}/create-github-page.rst>`_.

My local set-up included a:

   * Git repo with content and configuration files
   * Git submodule with generated output to host on GitHub
   * Git submodule with pelican-plugins

After reading through an article on static site hosting with `S3 and CloudFlare <https://wsvincent.com/static-site-hosting-with-s3-and-cloudflare/>`_ for "pennies a month" (caution: not yet verified!) I decided this was something I wanted to do.

My reasons include:

* Git submodule for constantly regenerated output was a really ugly solution
* Can use SSL and generally more professional
* Learn more about S3 and CloudFlare

In this article I'll describe how to:

1. Setup S3
2. Modify Pelican repo to publish to S3
3. Setup CloudFront and Modify Nameservers
4. Redirect from GitHub with `jekyll-redirect-from <https://github.com/jekyll/jekyll-redirect-from>`_

To make this work you do need a registered domain. There are many options. I am using `CanSpace <https://www.canspace.ca/>`_ which is good for Canadians wanting a .ca domain.

Amazon S3
---------

First create an `Amazon S3 account <https://aws.amazon.com/s3/>`_. If you are new to AWS then you can get 12-months of 5GB storage & 20k get requests per month free. This should be more than enough for a personal site. After that there `is a cost <https://aws.amazon.com/s3/pricing/>`_ but it's reasonable.

Like Will Vincent, the author of the original article, I created two buckets with with just my domain "joshmorel.ca" and another with the "www" subdomain which redirects to the first. 

Static Site Bucket
******************

To achieve a similar affect first click the **Create Bucket**, enter your domain for **Bucket name**, and click **Next** through the rest of the screens until created.

.. image:: {filename}/images/s3-create-bucket.png
   :alt: image: Create S3 Bucket

Click on the bucket, then navigate to **Permissions** > **Bucket Policy** and use the policy below (with your actual domain, of course), essentially allowing anyone to get any object in the bucket (which is what we want), being sure to click **Save**.

.. code-block:: console

   {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Sid": "AddPerm",
                "Effect": "Allow",
                "Principal": "*",
                "Action": "s3:GetObject",
                "Resource": "arn:aws:s3:::yourdomain.com/*"
            }
        ]
    }

.. image:: {filename}/images/s3-bucket-policy.png
   :alt: image: Set S3 Bucket Policy

To enable static website hosting, go to **Properties**, click **Static website hosting**, select the **Use this bucket to host a website** option, enter ``index.html`` and ``404.html`` for **index document** and **Error document** respectively, then click **Save**. Note that you will need to use a 404.html template to serve a custom 404 page, which I won't be covered in this article.

.. image:: {filename}/images/s3-static-site.png
   :alt: image: Enable S3 Static site hosting


Redirect Bucket
***************

Optionally you can create a ``www.yourdomain.com`` bucket which simply redirects to ``yourdomain.com``.

To do so create a new bucket similarly to above with the ``www`` subdomain as the bucket name. Then open the bucket, navigate to **Properties**, click **Static website hosting**, select the **Redirect requests** option and enter your domain for the **Target bucket or domain**.

The next step is to publish your site to S3.

Publishing to S3
----------------

Although you can upload files manually to S3 this is a horribly inefficient option. Instead, I recommend using the `s3cmd <http://s3tools.org/s3cmd>`_ tool (also written in Python) combined with the Pelican `Makefile <http://docs.getpelican.com/en/stable/publish.html#make>`_. Using `Makefile` on Windows may be tricky. I'll stick an Addendum to this article on this. But the main instructions will only work smoothly with Linux (my preference, although I do use Windows too) or Mac.

Enable s3cmd Management
***********************

To use s3cmd you must both install s3cmd and set up AWS Identity and Access Management (IAM). To install s3cmd, on Ubuntu 16.04 I used `sudo apt install s3cmd`. For others, you can check your package manager or visit `the Download page for other options <http://s3tools.org/download>`_.

You also need credentials for remote management, created through IAM. Log on to the AWS console navigate to the IAM service. You can also add required credentials to an existing user, but assuming you don't have one, click **Users** then **Add user**.

.. image:: {filename}/images/s3-iam-user-create.png
   :alt: image: Create AWS user

Enter a meaningful name and check off **Programmatic access** then click **Next: Permissions**.

.. image:: {filename}/images/s3-iam-set-user-details.png
   :alt: image: Set AWS user details

Unless you have an appropriate group or user to copy permissions from, click **Attach existing policies directly**, filter or browse for **AmazonS3FullAccess**, place a check next to it, then click **Next: Review** and finally **Create user**.

.. image:: {filename}/images/s3-iam-set-user-permissions.png
   :alt: image: Set AWS user S3 permissions

After successful creation, download the .csv file or copy the **Access key ID** and **Secret access key**. Either way, it is important to keep these secret. These credentials can allow any agent to create and use nearly unlimited S3 buckets under your account as well as compromise existing buckets.

Run ``s3cmd --configure`` providing the necessary details. For more information about the different options visit the `s3cmd how-to <http://s3tools.org/s3cmd-howto>`_. For region codes to use when prompted for **Default Region** see http://docs.aws.amazon.com/general/latest/gr/rande.html#s3_region.

Great, you are now good to publish the blog to S3 using Pelican.

Publish to S3 with Pelican Makefile
***********************************

I'm going to use the Pelican Makefile for publishing. Previously I was doing site generation and debugging with only the ``pelican`` & ``python -m SimpleHTTPServer`` commands while publishing with ``git push``.  Now I will be using Makefile commands for everything some of which require the ``develop_server.sh`` script. To create these files, with focus on S3, run ``pelican-quickstart`` again providing the answering the S3-related prompts appropriately (lines beginning ``>`` below):

.. code-block:: bash

   pelican-quickstart
   > Do you want to upload your website using S3? (y/N) y
   > What is the name of your S3 bucket? [my_s3_bucket] yourdomain.com
   mv Makefile develop_server.sh path/to/your/siterepo

Note: If your content and output sub-directories are not named ``content`` or ``output`` then you will need to edit these files.

To publish your content to S3:

.. code-block:: bash

   cd path/to/your/siterepo
   make publish
   make s3_upload

You should now be able to see your see your site index file at your bucket endpoint, for example: http://s3.ca-central-1.amazonaws.com/yourdomain.com/index.html. But this is less than ideal. We want to be able to use our own domain name and SSL. For this we can use a free tier of CloudFlare.

Leveraging CloudFlare for Secure Site Delivery
----------------------------------------------

CloudFlare provides improved content delivery, security and domainname services with a free-tier that should be sufficient for our purposes.





