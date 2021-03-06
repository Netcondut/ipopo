.. Shell Service tutorial

Shell Service
#############

Description
***********

In order to interact with the Pelix framework and its components, the shell
service allows to register a set of commands that will be used by shell
interfaces.

The core shell module stores the command regrouped into namespaces.
The name to use in a console to access a command is ``namespace.command``.

Pelix/iPOPO distribution comes with four bundles:

+------------------------+-----------------------------------------------+
| Bundle                 | Description                                   |
+========================+===============================================+
| pelix.shell.core       | Provides the core shell service, the commands |
|                        | registry and the basic Pelix commands         |
+------------------------+-----------------------------------------------+
| pelix.shell.ipopo      | Provides iPOPO shell commands                 |
+------------------------+-----------------------------------------------+
| pelix.shell.console    | An interactive console based on *readline*    |
+------------------------+-----------------------------------------------+
| pelix.shell.remote     | Provides a raw-TCP access to the shell        |
+------------------------+-----------------------------------------------+
| pelix.shell.eventadmin | Provides commands to send or post events      |
|                        | (see :ref:`eventadmin`)                       |
+------------------------+-----------------------------------------------+


Usage
*****

Using the interactive console
=============================

The ``pelix.shell.console`` module can be used as standard bundle and
installed in a framework instance, but it can also be used as a script to
start a framework instance, with only iPOPO and the shell bundles pre-installed.

For example:

.. code-block:: none
   :linenos:
   
   $ python -m pelix.shell.console
   ** Pelix Shell prompt **
   $ bl
   +----+---------------------+--------+-----------+
   | ID |        Name         | State  |  Version  |
   +====+=====================+========+===========+
   | 0  | org.psem2m.pelix    | ACTIVE | (0, 4, 0) |
   +----+---------------------+--------+-----------+
   | 1  | pelix.shell.core    | ACTIVE | (0, 1, 0) |
   +----+---------------------+--------+-----------+
   | 2  | pelix.shell.console | ACTIVE | (0, 1, 0) |
   +----+---------------------+--------+-----------+
   | 3  | pelix.shell.ipopo   | ACTIVE | (0, 1, 0) |
   +----+---------------------+--------+-----------+
   $ quit
   Bye !


.. note:: Pelix must be correctly installed (i.e. accessible in your Python
   path) for this command to work.


The commands that you can execute are described in the section
:ref:`shell-commands`.

If you are running the script on a platform that provides the ``readline``
module (most of Unix platforms), the interactive console can complete command
names using the *tab* key.


Instantiation
=============

The Pelix core shell service and the iPOPO shell commands are available as
soon as their respective bundle is started.

.. code-block:: python
   :linenos:
   
   from pelix.framework import FrameworkFactory
   
   # Start the framework
   framework = FrameworkFactory.get_framework()
   framework.start()
   context = framework.get_bundle_context()
   
   # Install & start iPOPO
   context.install_bundle('pelix.ipopo.core').start()
   
   # Install & start the Pelix core shell
   context.install_bundle('pelix.shell.core').start()
   
   # Install & start the iPOPO commands
   context.install_bundle('pelix.shell.ipopo').start()


The remote shell must be instantiated using iPOPO, using the
``ipopo-remote-shell-factory`` factory:

.. code-block:: python
   :linenos:

   # Get the iPOPO service
   from pelix.ipopo.constants import get_ipopo_svc_ref
   ipopo = get_ipopo_svc_ref(context)[1]
   
   # Install & start the remote shell bundle
   context.install_bundle('pelix.shell.remote').start()
   
   # Instantiate a remote shell
   ipopo.instantiate('ipopo-remote-shell-factory', 'ipopo-remote-shell')


By default, the remote shell listens on port 9000, you can access it using
softwares like *telnet* or *netcat*.


Configuration
=============

The core shell service and the iPOPO commands component are not configurable.

The remote shell component can be configured using the following properties:

+---------------------+---------------+--------------------------------------+
| Property            | Default value | Description                          |
+=====================+===============+======================================+
| pelix.shell.address | localhost     | Address the server will be bound to  |
+---------------------+---------------+--------------------------------------+
| pelix.shell.port    | 9000          | TCP port that the server will listen |
|                     |               | to                                   |
+---------------------+---------------+--------------------------------------+


Interface
=========

Core shell service
------------------

The core shell service provides the following interface:

+---------------------------------+--------------------------------------------+
| Method                          | Description                                |
+=================================+============================================+
| register_command(namespace,     | Associates the given method to the given   |
| command, method)                | name in the given name space               |
+---------------------------------+--------------------------------------------+
| unregister(namespace, command)  | Unregister the given command from the      |
|                                 | given name space, or the whole name space  |
|                                 | if command is None                         |
+---------------------------------+--------------------------------------------+
| execute(cmdline, stdin, stdout) | Parses and executes the given command line |
|                                 | with given input and output streams        |
+---------------------------------+--------------------------------------------+
| get_banner()                    | Retrieves the welcome banner for the shell |
+---------------------------------+--------------------------------------------+
| get_ps1()                       | Retrieves the prompt string                |
+---------------------------------+--------------------------------------------+


Utility shell service
---------------------

The utility shell service can be used to ease commands implementations.
It provides the following methods:

+----------------------------+----------------------------------------------+
| Method                     | Description                                  |
+============================+==============================================+
| bundlestate_to_str(state)  | Retrieves the string representation of the   |
|                            | state of a bundle                            |
+----------------------------+----------------------------------------------+
| make_table(headers, lines) | Generates an ASCII table using the given     |
|                            | column headers (N-tuple) and the given lines |
|                            | (array of N-tuples)                          |
+----------------------------+----------------------------------------------+


Command method
--------------

A command method must accept an ``IOHandler`` object as its first parameter and
must use it to interact with the client.
The remote shell is based on this behavior, given the client socket as the
input and output of the commands to execute.

Also, a command method should have a documentation, that will be used as its
help message.

Here is the implementation of the *start* method, which starts a bundle with
the given ID:

.. code-block:: python
   :linenos:
   
   def start(self, io_handler, bundle_id):
        """
        start <bundle_id> - Starts the given bundle ID
        """
        bundle_id = int(bundle_id)
        bundle = self._context.get_bundle(bundle_id)
        if bundle is None:
            io_handler.write_line("Unknown bundle: {0}", bundle_id)

        bundle.start()


Command service
---------------

The core shell service automatically registers all services providing the
``pelix.shell.command`` specification.

Those services must implement the following methods:

+---------------------+-----------------------------------------------------+
| Method              | Description                                         |
+=====================+=====================================================+
| get_namespace()     | Retrieves the name space of the provided commands   |
+---------------------+-----------------------------------------------------+
| get_methods()       | Retrieves the list of (command, method) tuples      |
+---------------------+-----------------------------------------------------+
| get_methods_names() | Retrieves the list of (command, method name) tuples |
+---------------------+-----------------------------------------------------+

The ``get_methods_names()`` method is here to prepare remote services tests,
and will allow to execute commands from a distant framework.


.. _shell-commands:

Commands
********

Core
====

These commands are in the name space ``default``, they can be called without
specifying it.

+-------------------+-----------------------------------------+
| Command           | Description                             |
+===================+=========================================+
| help, ?           | Prints the registered shell commands    |
+-------------------+-----------------------------------------+
| quit, exit, close | Exits the shell sessions                |
+-------------------+-----------------------------------------+
| bd <ID>           | Prints the details of the given bundle  |
+-------------------+-----------------------------------------+
| bl                | Prints the list of installed bundles    |
+-------------------+-----------------------------------------+
| sd <ID>           | Prints the details of the given service |
+-------------------+-----------------------------------------+
| sl                | Prints the list of registered services  |
+-------------------+-----------------------------------------+
| start <ID>        | Starts the bundle with the given ID     |
+-------------------+-----------------------------------------+
| stop <ID>         | Stops the bundle with the given ID      |
+-------------------+-----------------------------------------+
| update <ID>       | Updates the bundle with the given ID    |
+-------------------+-----------------------------------------+
| install <name>    | Installs the bundle with the given name |
+-------------------+-----------------------------------------+
| uninstall <ID>    | Uninstalls the bundle with the given ID |
+-------------------+-----------------------------------------+


iPOPO
=====

These commands are in the name space ``ipopo`` and needs the
``pelix.ipopo.core`` service to be registered, which means that the bundle
``pelix.ipopo.core`` must be installed.

+------------------------------+--------------------------------------------+
| Command                      | Description                                |
+==============================+============================================+
| factories                    | Prints the registered factories            |
+------------------------------+--------------------------------------------+
| instances                    | Prints the instantiated components         |
+------------------------------+--------------------------------------------+
| instance <name>              | Prints the details of the given component  |
|                              | instance                                   |
+------------------------------+--------------------------------------------+
| instantiate <factory> <name> | Instantiate the component of the given     |
| [<property=value> [...]]     | factory with the given name and properties |
+------------------------------+--------------------------------------------+
| kill <name>                  | Kills the component of the given name      |
+------------------------------+--------------------------------------------+


Sample
======

Here is a sample usage of the remote shell, using *netcat* (*nc*) for the
connection and *rlwrap* to allow line modifications:

.. code-block:: none
   :linenos:
   
   
   $ rlwrap nc localhost 9000
   ------------------------------------------------------------------------
   ** Pelix Shell prompt **
   iPOPO Remote Shell
   ------------------------------------------------------------------------
   $ bl
   +----+--------------------+--------+-----------+
   | ID |        Name        | State  |  Version  |
   +====+====================+========+===========+
   | 0  | org.psem2m.pelix   | ACTIVE | (0, 4, 0) |
   +----+--------------------+--------+-----------+
   | 1  | pelix.ipopo.core   | ACTIVE | (0, 4, 0) |
   +----+--------------------+--------+-----------+
   | 2  | pelix.shell.core   | ACTIVE | (0, 1, 0) |
   +----+--------------------+--------+-----------+
   | 3  | pelix.shell.ipopo  | ACTIVE | (0, 1, 0) |
   +----+--------------------+--------+-----------+
   | 4  | pelix.shell.remote | ACTIVE | (0, 1, 0) |
   +----+--------------------+--------+-----------+
   $ sl
   +----+---------------------------+--------------------------------------+---------+
   | ID |      Specifications       |                Bundle                | Ranking |
   +====+===========================+======================================+=========+
   | 1  | ['pelix.ipopo.core']      | Bundle(ID=1, Name=pelix.ipopo.core)  | None    |
   +----+---------------------------+--------------------------------------+---------+
   | 2  | ['pelix.shell']           | Bundle(ID=2, Name=pelix.shell.core)  | None    |
   +----+---------------------------+--------------------------------------+---------+
   | 3  | ['pelix.shell.utilities'] | Bundle(ID=2, Name=pelix.shell.core)  | None    |
   +----+---------------------------+--------------------------------------+---------+
   | 4  | ['ipopo.shell.command']   | Bundle(ID=3, Name=pelix.shell.ipopo) | None    |
   +----+---------------------------+--------------------------------------+---------+
   $ ipopo.instances
   +----------------------+------------------------------+------------+
   |         Name         |           Factory            |   State    |
   +======================+==============================+============+
   | ipopo-remote-shell   | ipopo-remote-shell-factory   | VALIDATING |
   +----------------------+------------------------------+------------+
   | ipopo-shell-commands | ipopo-shell-commands-factory | VALID      |
   +----------------------+------------------------------+------------+
   $ 


How to write a command provider
*******************************

This snippet shows how to write a component providing the command service:

.. code-block:: python
   :linenos:
   
   from pelix.ipopo.decorators import ComponentFactory, Provides, Instantiate
   
   @ComponentFactory(name='simple-command-factory')
   @Instantiate('simple-command')
   @Provides(specifications='pelix.shell.command')
   class SimpleServletFactory(object):
       """
       Simple command factory
       """
       def __init__(self):
           """
           Set up the component
           """
           self.counter = 0
       
       def get_namespace(self):
           """
           Retrieves the commands name space
           """
           return "counter"
       
       def get_methods(self):
           """
           Retrieves the commands - methods association
           """
           return [("more", self.increment),
                   ("less", self.decrement),
                   ("print", self.print)]
       
       def get_methods_names(self):
           """
           Retrieves the list of tuples (command, method name) for this command
           handler.
           """
           result = []
           for command, method in self.get_methods():
               result.append((command, method.__name__))

           return result

           
       def increment(self, io_handler, value=1):
           """
           Increments the counter of [value]
           """
           self.counter += value
       
       
       def decrement(self, io_handler, value=2):
           """
           Decrements the counter of [value]
           """
           self.counter -= value
       
       
       def print(self, io_handler):
           """
           Prints the value of the counter
           """
           io_handler.write_line('Counter = {0}', self.counter)


Now you can install this bundle and use the commands *counter.more*,
*counter.less* and *counter.print*.
