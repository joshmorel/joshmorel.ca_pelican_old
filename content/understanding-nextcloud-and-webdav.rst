Understanding Nextcloud and WebDAV
##################################
:date: 2017-02-20 15:38
:modified: 2017-02-20 15:38
:tags:
:status: draft
:category: Household cloud
:slug: understanding-nextcloud-and-webdav
:authors: Josh Morel
:summary: Some additional explanations on Nextcloud and it's use of WebDAV and related technology.
:series: Nextcloud

.. role:: console(code)
   :language: console

Background
----------

Before moving on to my next series I'm going to provide some understanding of how the Nextcloud server and clients work. Many times its fine and even necessary to call take advantage of abstraction and call some underlying technology magic. But often times, especially when you are responsible for managing that thing, deep understanding is important.

So I'm going to do that with WebDAV specifically within the context of Nextcloud and its clients.

WebDAV
------

Short for Web Distributed Authoring and Versioning, `WebDAV <https://en.wikipedia.org/wiki/WebDAV>`_ is a protocol which extends the Hypertext Transfer Protocol (HTTP). HTTP itself has made the web possible but in my opinion it is WebDAV which makes the web a truly collaborative place for all users.

Prior to WebDAV, the web was practically a "read-only" thing for your average user. Of course, the original vision of this was not The need for something different

The protocol actually dates back to 1999 with `Request for Comment (RFC) 2518 <https://tools.ietf.org/html/rfc2518>`_ from the Internet Engineer Task Force (IETF). The 2nd and active version was released in 2007 with `RFC 4918 <https://tools.ietf.org/html/rfc4918>`_.

The need for this protocol however was outlined comprehensively in `RFC 2291 <https://tools.ietf.org/html/rfc2291>`_ titled "Requiredments for a Distributed Authoring and Versioning Protocol for the World Wide Web".


WebDAV in Nextcloud
~~~~~~~~~~~~~~~~~~~

Architecture!

WebDAV Server
~~~~~~~~~~~~~

