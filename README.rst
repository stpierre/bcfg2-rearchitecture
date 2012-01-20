==========
 Overview
==========

This describes my plans for a rearchitecture of the Bcfg2 server to
make it far more scalable.  Plans in this document are derived from my
discussions with many Bcfg2 developers and users in person, on IRC,
and on the bcfg-dev mailing list.

My plan is to split up the server into a few basic components which,
with proper asynchronous message passing, could be run on a single
machine or on multiple machines to let people scale where their
bottlenecks are.  The components are:

#. A very lightweight "master" process that handles SSL,
   authentication, and request routing.  It would also handle routing
   portions of the initial portion of the client-server discussion --
   getting probes, receiving probe data, and getting decision lists.
#. A configuration render process that does all the actual heavy
   lifting of rendering templates, generating package lists, and so on.
   This is most people's bottleneck nowadays, so I figure most people
   would want to run multiple renderers.
#. A statistics processing process to process and store statistics.
   This is currently threaded separately from the main Bcfg2 server
   process, so splitting it off is natural.
#. A database to store data that is managed exclusively by the Bcfg2 server
#. Asynchronous message passing between all three components.

==============
 Server Types
==============

Master servers handle:

* SSL
* Authentication
* XML-RPC
* Routing of requests

Configuration renderers handle:

* Rendering templates
* Gathering content from the content servers and compiling it into a
  bound specification
* FAM
* Plugins that touch the repo (Properties write-back, Svn2, etc.)
* Building and storing metadata

Statistics processors handle:

* Processing stats

The database handles:

* Storage of all data that is managed exclusively by the Bcfg2
  server (E.g., probed.xml, Ohai data, client list)
* Optionally, results from the AMQP queues

The VCS handles:

* Storage of all data that is managed exclusively by the user
* Storage of data that is created by the Bcfg2 server, but can be
  managed by the user (e.g., SSHbase, SSLCA, NagiosGen, etc.)

clients.xml
-----------

Most of the data Bcfg2 uses is fairly obviously either maintained
entirely by the server (e.g., ``probed.xml``, the record of Probe
data), and thus stored in the database; or maintained entirely by the
user (e.g., most things in Cfg), and thus stored in the VCS; or
initially created by the server but customizable by the user (e.g.,
SSHbase data), and thus stored in the VCS.

The one exception to this is ``clients.xml``, which is created by the
user and updated by both the server and the user on an ongoing basis.
``clients.xml`` actually handles three separate tasks:

* It holds a full list of all clients, which is automatically added to
  by the server;
* It specifies profile groups for each client, which are set by the
  user; and
* It specifies detailed authentication information for each client,
  which may be set by the user.

In order to handle splitting up the database and VCS portions of data,
and splitting the authentication process off from the machines that
actually hold the Bcfg2 data, we will divide the three tasks of
``clients.xml``:

* The full client list will be held in the database, and updated as
  necessary by the configuration renderers.
* ``Metadata/clients.xml`` will be used to set profile groups.  A
  completely empty ``clients.xml`` is perfectly reasonable if profile
  groups are not in use (e.g., if GroupPatterns are used instead).
* ``authentication.xml``, a file that will live on the master
  server(s) and will not be part of the normal Bcfg2 repository, will
  be used to customize the way individual clients authenticate.

Since ``clients.xml`` will be the only way to specify a profile group,
and since it will no longer by modifiable by the Bcfg2 server, this
removes the ability of the Bcfg2 server to assert a profile group.
Consequently, the ``-p`` flag to ``bcfg2`` will be removed.  Profile
groups will have to be changed in ``clients.xml`` by editing the file.

==============
 Interactions
==============

Normal Client-Server Interaction
--------------------------------

#. Client announces version to master (6789/tcp)
#. Client requests probes from master (6789/tcp)
#. Master requests probes from configuration queue (5672/tcp). A
   renderer:
   #. Builds initial metadata for client
   #. Creates probe set
   #. Returns probes to master (5672/tcp)

#. Master returns probes to client (6789/tcp)
#. Client:
   #. Runs probes
   #. Returns probe data to master (6789/tcp)

#. Master sends probe data to configuration queue (5672/tcp)
#. A renderer processes the probe data.
#. Client requests decision list from master (6789/tcp)
#. Master requests decision list from configuration queue
   (5672/tcp). A renderer:
   #. Builds full metadata for client
   #. Creates decision list
   #. Returns decision list to master (5672/tcp)

#. Master returns decision list to client (6789/tcp)
#. Client requests full configuration from master (6789/tcp)
#. Master submits configuration render request to configuration queue
   (5672/tcp)
#. Configuration renderer:
   #. Builds full metadata for client
   #. Builds structure list
   #. Validates structures
   #. Binds structures with
   #. Returns fully-bound configuration to master (5672/tcp)

#. Master sends configuration to client (6789/tcp)
#. Client applies configuration
#. Client sends statistics to master server (6789/tcp)
#. Master server submit statistics to stats processing queue
   (5672/tcp)
#. Statistics processor processes statistics from queue

XML-RPC
-------

#. Client sends XML-RPC request to master (6789/tcp)
#. Master determines if XML-RPC request applies to _every_ renderer or
   to _any_ renderer.
#. If the request is one-to-all:
   #. The master routes the request to the XML-RPC queue (5672/tcp).
   #. All renderers process the request.
   #. The master returns ok.

#. If the request is one-to-any:
   #. The master routes the request to the configuration render queue
      (5672/tcp).
   #. One renderer processes the request, and submits the results to
      the configuration result queue (5672/tcp).
   #. The master returns the result.

To distinguish between one-to-any and one-to-every XML-RPC requests,
the existing ``__rmi__`` Plugin class variable will be extended.
Entries to the ``__rmi__`` list can be:

* PluginRMI objects, which are simple structs that have "method",
  "rmi_type", and "public" attributes; or
* Plain strings for backwards compatibility.

For instance, you might have::

  class Plugin(object):
      __rmi__ = [PluginRMI("toggle_debug",
                           rmi_type=PluginRMI.one_to_all)]

Or::

  class Probing(object):
      __rmi__ = [PluginRMI("ReceiveDataItem",
                           rmi_type=PluginRMI.one_to_any,
                           public=False)]

===========
 Protocols
===========

All protocols use JSON objects to pass their data.

Configuration Queue
-------------------

The Configuration Queue is an AMQP qork queue called "configuration".
The masters make rendering requests, which are satisfied by individual
configuration rendering servers.

A rendering request consists of a single line::

  "<hostname>"

Those are literal quotes; remember, this is a JSON object.

The results are passed to the results backend as XML documents.

One-to-Any XML-RPC Queue
------------------------

The One-to-Any XML-RPC Queue processes One-to-Any XML-RPC requests.
It is an AMQP qork queue called "anyrpc".

An XML-RPC request consists of a JSON object representing a dict with
the keys ``plugin``, ``method``, and ``args``.  For instance::

  {"args": [], "method": "Update", "plugin": "Svn2"}

Or::

  {"args": ["foo.example.com", "<Probe name=\"test\">group:test</Probe>"],
   "method": "ReceiveDataItem",
   "plugin": "Probes"}

The result will be passed to the results backend as a JSON object
that will be loaded and returned as the XML-RPC reply.

Note that this does add the ability to handle arguments to XML-RPC
calls internally; bcfg2-admin would need modifications to support that
externally as well.

Probe data processing will both be handled as if it were an XML-RPC
call to Probes.ReceiveDataItem.

One-to-All XML-RPC Queue
------------------------

The One-to-All XML-RPC Queue processes One-to-All XML-RPC requests.
It is an AMQP publish/subscribe queue called "allrpc".

Requests and replies are in the same format as `One-to-Any XML-RPC
Queue`_ requests.

=======================
 Sample Configurations
=======================

A small environment
-------------------

One goal is to keep small environments simple.  A small, simple
environment could be entirely hosted on a single box.

The machine in question would run the master server, which would use
AMQP messages on localhost (over a RabbitMQ server running on the
selfsame machine) to communicate with the configuration rendering
processes and statistics processor.  Depending on the size of the
machine, you might choose to run two renderers and one statistics
processor.

Bcfg2 server data would be stored in a local SQLite database; user
data would be stored in a VCS repository hosted on the same machine
and checked out locally into ``/var/lib/bcfg``.

A large environment
-------------------

At the opposite end, we want to make it easy to scale to very large
environments.  Every component can be made fully redundant and split
out from the others.

On the front end, a pair of small master servers would handle request
routing.  The most resource-intensive part of Bcfg2, rendering the
configuration, would be handled by a farm of four large rendering
machines each running 16 rendering processes.  Each physical machine
would have the VCS repository, hosted on a separate, highly-available
cluster, checked out into a tmpfs volume to maximize speed.  Bcfg2
server data would be stored in a scalable NewSQL database.

Statistics processing would be handled by another pair of small
machines, each running four statistics processes.

Queuing would be done on a highly available RabbitMQ cluster, and
results would be stored in a memcached cluster.

This sort of very large deployment could easily cover 20 machines, or
even more.  Since the topology is highly distributed, it is easy to
scale where your needs lie.

==============
 Technologies
==============

Much has been said about the general architecture of the Bcfg2
service, but we have mostly avoided mention of specific technologies.
These are discussed below.  I have also included some brief discussion
of other possibilities I considered but rejected.

Master Servers
--------------

The master server will be implemented as a WSGI script suitable for
use by Apache, Nginx, or another web server capable of executing WSGI
scripts.  This garners several wins:

* SSL is completely free, as it will be implemented by the web server
* Traditionally, WSGI scripts themselves are stateless, so, by using
  WSGI and by not allowing a connection from the master server to the
  database or VCS, we enforce the statelessness (and thus the light
  weight) of the master server.
* Threading is completely free

Other options considered
~~~~~~~~~~~~~~~~~~~~~~~~

* Mongrel2: Originally considered when I was considering ZeroMQ for
  the message passing fabric.  Without a general requirement for
  ZeroMQ, using Mongrel2 unnecessarily introduces excessive
  dependencies.
* Twisted: Writing a reactor in Twisted would get us SSL and
  authentication quite cheaply, but not entirely free.  It would,
  however, be a stateful server, and it would be asynchronous rather
  than threaded; given how lightweight the server is, I doubt that
  would make much difference in performance, but I imagine there would
  be some, and the threaded model is likely to come out on top.

Configuration Renderers
-----------------------

The configuration renderers will run custom daemons based on the
current ``bcfg2-server`` core code.

Statistics Processors
---------------------

The statistics processors will run custom daemons based on a very
small subset of the current ``bcfg2-server`` core code.

Message Passing
---------------

Bcfg2 will use the Celery library as its message passing interface.
By default, we will use RabbitMQ for message passing and the AMQP
results backend, but other options are configurable by the end user.

Other options considered
~~~~~~~~~~~~~~~~~~~~~~~~

* ZeroMQ: Very flexible, is implemented at a much lower level than
  Celery/AMQP, so was determined to be overkill.  Additionally,
  implementing a simple AMQP worker-style queue in ZeroMQ turns out to
  be quite difficult, and high availability of such a queue is
  nontrivial.
* Other default backends: The SQLAlchemy message transport backend to
  Celery has several limitations, most damning of which is a limit of
  only a few worker nodes. The database results backend is not so
  limited, but using the AMQP backend keeps our message passing
  consistent.  The only other backends that can be used for both
  message transport and results storage are Redis and MongoDB, and
  these both unnecessarily introduce extra dependencies.

Database
--------

We will use SQLAlchemy to interface with the database, allowing the
user to pick any supported relational database they wish.

VCS
---

We will aim to provide support for all VCS systems Bcfg2 currently has
version plugins for: Bazaar, CVS, Darcs, Fossil, Git, Mercurial, and
Subversion.

==========
 Diagrams
==========

I have included three diagrams to help visualize the architecture:

* ``bcfg2-overview.gv`` is a broad overview of the communication that
  occurs for a normal client request
* ``bcfg2-xml-rpc.gv`` is an overview of the communication that occurs
  for an XML-RPC request
* ``bcfg2-tech.gv`` is a diagram of the various technologies at work

These can be rendered with::

  dot -T png <gv-file> > output.png
