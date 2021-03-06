.. Remote Services

.. _remote_services:

Remote Services
###############

Description
***********

Remote Services is a set of bundles that provides an easy-to-use way to
provide or use services from another framework, either in a different process
or on a different machine.

It is inspired from the OSGi Remote Services specification, and reuses its
properties naming.

Core
====

Two bundles contain the *core* of the remote services implementation:

* ``pelix.remote.dispatcher``, which contains the registry of exported services,
  and defines utility methods for the transport layer
* ``pelix.remote.registry``, which contains the registry of imported services

Both automatically instantiate their core components.

Dispatcher servlet
------------------

The dispatcher servlet is an *optional* component that provides a servlet to
be installed in an HTTP service.
It allows remote framework to retrieve information about exported services.

The multicast discovery provided with Pelix, described below, requires this
servlet.

The ``pelix.remote.dispatcher`` bundle defines the dispatcher component, which
is not automatically instantiated.
Its factory, ``pelix-remote-dispatcher-servlet-factory`` accepts the following
configuration property:

+-----------------+-------------------+-----------------------------------+
| Property        | Default value     | Description                       |
+=================+===================+===================================+
| pelix.http.path | /pelix-dispatcher | The path to access the dispatcher |
|                 |                   | servlet                           |
+-----------------+-------------------+-----------------------------------+


Discovery
=========

The discovery bundles are here to detect the other frameworks and their exported
services, and to notify other frameworks of their own services events.

When it detects a remote service, the discovery bundle will retrieve its
description from the remote framework and give it to the registry of imported
service.
The latter will notify the transport implementations of this discovery and let
them handle the service registration, if any.

When a service is declared exported, the dispatcher will notify the discovery
implementations, which shall propagate this event to other frameworks.

Multiple discovery implementations can run in the same framework.

Multicast
---------

.. important:: This implementations depends on the HTTP service, and the
   dispatcher servlet must be registered.

Pelix comes with a home-made UDP multicast discovery protocol, implemented in
the ``pelix.remote.discovery.multicast`` bundle.
It defines a ``pelix-remote-discovery-multicast-factory`` iPOPO factory, which
is not automatically instantiated.

This factory accepts the following configuration properties:

+-----------------+---------------+------------------------------------------+
| Property        | Default value | Description                              |
+=================+===============+==========================================+
| multicast.group | 239.0.0.1     | The multicast group (address) to join to |
|                 |               | send and receive discovery messages.     |
+-----------------+---------------+------------------------------------------+
| multicast.port  | 42000         | The multicast port                       |
+-----------------+---------------+------------------------------------------+

.. important:: Do not forget to configure the firewall to accept IGMP packets
   (multicast setup packets) and to open the UDP port used in "multicast.port"

Transport
=========

A transport implementation bundle should be separated in two parts (or two
components):

* the service exporter:

  * detects the services to export,
  * creates a proxy or registers it in a transport-level dispatcher
  * tells the core dispatcher that it exports a service

* the service importer:

  * notified by the registry of imported services
  * checks if it can handle the discovered service
  * creates a proxy
  * registers the proxy as a local service

Multiple transport implementations can run in the same framework.

XML-RPC
-------

.. important:: This implementations depends on the HTTP service
.. note:: XML-RPC has several limitations due to the ``xmlrpclib``, especially
   about nested dictionaries.

The XML-RPC transport implementation, in the ``pelix.remote.xml_rpc`` bundle,
is based on the ``xmlrpclib`` standard module.
It defines two iPOPO components, the importer and the exporter, that are not
automatically instantiated.

* the exporter factory is ``pelix-xmlrpc-exporter-factory``.
  It accepts the following configuration property:

  +-----------------+---------------+--------------------------------+
  | Property        | Default value | Description                    |
  +=================+===============+================================+
  | pelix.http.path | /XML-RPC      | The path to access the XML-RPC |
  |                 |               | dispatcher                     |
  +-----------------+---------------+--------------------------------+

* the import factory is ``pelix-xmlrpc-importer-factory``.
  It does not need configuration properties.


JSON-RPC
--------

.. important:: This implementations depends on the HTTP service
.. important:: The widely used ``jsonrpclib`` does not work with this
   implementation, as it does not allow custom dispatch methods. As both the
   original and the patched version use the same package name, ``jsonrpclib``,
   it is necessary to remove the original version first.

The JSON-RPC transport implementation, in the ``pelix.remote.json_rpc`` bundle,
is based on a patched version of the ``jsonrpclib`` package.

You can use the following command to install this package:

.. code-block:: console

   easy_install -U jsonrpclib-pelix
   # or
   pip install jsonrpclib-pelix

or you can download it from
`github.com/tcalmant/jsonrpclib <https://github.com/tcalmant/jsonrpclib>`_,
then using the command:

.. code-block:: console

   python setup.py install


The ``pelix.remote.json_rpc`` bundle defines two iPOPO components, the importer
and the exporter, that are not automatically instantiated.

* the exporter factory is ``pelix-jsonrpc-exporter-factory``.
  It accepts the following configuration property:

  +-----------------+---------------+--------------------------------+
  | Property        | Default value | Description                    |
  +=================+===============+================================+
  | pelix.http.path | /JSON-RPC     | The path to access the XML-RPC |
  |                 |               | dispatcher                     |
  +-----------------+---------------+--------------------------------+

* the import factory is ``pelix-jsonrpc-importer-factory``.
  It does not need configuration properties.


Usage
*****

Installation
============

Pelix remote services implementation needs at least the core bundles, a
discovery bundle and a transport bundle to work.

In this snippet, we install and instantiate the multicast discovery and the
JSON-RPC transport:

.. code-block:: python
   
   # Pelix
   import pelix.framework
   from pelix.ipopo.constants import get_ipopo_svc_ref

   BUNDLES = ("pelix.ipopo.core",
              "pelix.shell.core",
              "pelix.shell.ipopo",
              "pelix.shell.console",
              "pelix.shell.eventadmin",
              "pelix.http.basic",
              "pelix.remote.dispatcher",
              "pelix.remote.registry",
              "pelix.remote.json_rpc",
              "pelix.remote.discovery.multicast",
              "pelix.services.eventadmin")
   """ Bundles to install by default in the Pelix framework """
   
   # Prepare the framework + iPOPO + shell
   framework = pelix.framework.create_framework(BUNDLES)
   context = framework.get_bundle_context()
   
   # Start it
   framework.start()
   
   # Instantiate components...
   # ... HTTP Service
   ipopo.instantiate("pelix.http.service.basic.factory",
                     "pelix.http.service.basic",
                     {"pelix.http.port": 8080})
                     
   # ... dispatcher servlet
   ipopo.instantiate("pelix-remote-dispatcher-servlet-factory",
                     "pelix-remote-dispatcher-servlet", {})
   
   # ... multicast discovery
   ipopo.instantiate("pelix-remote-discovery-multicast-factory",
                     "pelix-remote-discovery-multicast", {})

   # ... JSON-RPC exporter and importer
   ipopo.instantiate("pelix-jsonrpc-exporter-factory",
                     "pelix-jsonrpc-exporter", {})
   ipopo.instantiate("pelix-jsonrpc-importer-factory",
                     "pelix-jsonrpc-importer", {})


Export a service
================

A service must be exported if it has one of the following properties:

+-----------------------------+---------+-------------------------------------+
| Property                    | Type    | Description                         |
+=============================+=========+=====================================+
| service.exported.configs    | List of | A list of transport names.          |
|                             | strings | Transports that do not handle this  |
|                             |         | name must ignored this service.     |
|                             |         | This allows transport-specific      |
|                             |         | features to be used.                |
+-----------------------------+---------+-------------------------------------+
| service.exported.interfaces | List of | The list of services specifications |
|                             | strings | to export. All specifications are   |
|                             | or '*'  | exported if this property is        |
|                             |         | missing.                            |
|                             |         | This allows to export only a part   |
|                             |         | of the services provided by a       |
|                             |         | component.                          |
+-----------------------------+---------+-------------------------------------+

Here is a sample component that export its service:

.. code-block:: python

   @ComponentFactory("hello-world-factory")
   @Provides("hello.world")
   @Requires('_event', pelix.services.SERVICE_EVENT_ADMIN)
   @Property('_export_interface', pelix.remote.PROP_EXPORTED_INTERFACES,
             ["hello.world"])
   @Instantiate('hello-world')
   class HelloWorld(object):
       """
       Remote Hello World
       """
       def __init__(self):
           """
           Sets up the probe
           """
           # Export properties
           self._export_config = None
           self._export_interface = None

       def hello(self):
           """
           Classic method
           """
           print('Hello, world')

           
.. note:: The import of a service is transparent for its consumers
