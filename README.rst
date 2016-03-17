==========
TwiceRedis
==========
read and write sentinel connection pool backed redis client
with disconnecting sentinel clients and redis clients

when using a redis connection backed by a ``SentinelConnectionPool``,
the pool is initialized to connect to either the master or slaves.
this is annoying if you want to read from slaves and occasionally
write to the master.
``TwiceRedis`` solves this by having two clients,
one for master and one for slaves:

* ``tr.master`` also aliased as ``tr.write``
* ``tr.slave`` also aliased as ``tr.read``

these clients are each backed by a separate ``SentinelConnectionPool``
initialized to connect to the master or slaves respectively

``TwiceRedis`` also uses a ``DisconnectingSentinel`` class to drastically
reduce the number of active connections to the redis sentinel service(s).
This class drops connection to the chosen sentinel once the master or
slave has been chosen

The ``DisconnectionSentinel`` class also filters slaves a little more
intelligently than the base ``Sentinel`` class does. In addition to
insuring slaves are not ``sdown`` or ``odown`` it makes sure the slaves
``master-link-status`` is 'ok'

``TwiceRedis`` randomizes the sentinel list so each ``TwiceRedis``
object will be connecting to a random sentinel in the list instead of
them all connecting to the first one (for as long as it works).
this shuffling is probably a bit superfluous used in conjunction with
``DisconnectingSentinel``, but at worst will reduce the load on the
first sentinel in the ``sentinels`` list

``TwiceRedis`` also utilizes a subclass of ``StrictRedis`` called
``DisconnectRedis`` that adds a ``disconnect()`` function to all the clients
making it easier to manage individual connections to the redis services


~~~~~
usage
~~~~~
.. code:: python

    from twiceredis import TwiceRedis
    sentinels = [('10.10.10.10', 26379),
                 ('10.10.10.11', 26379),
                 ('10.10.10.12', 26379)]
    tr = TwiceRedis('master01', sentinels, 'tötes_passowrd')
    x = tr.slave.get('superkey')
    tr.master.set('je mange', 'huehue')
    x = tr.read.get('nous mangeons')
    tr.write.del('superkey')

pipelines work great too, you just have decide whether you need to write
during one or not, if write is needed, use ``tr.master``, else use ``tr.slave``

.. code:: python

    with tr.master.pipeline() as wpipe:
        wpipe.set('turle', 'power')
        wpipe.set('tr3buchet', 'tötes')
        wpipe.execute()

to connect, get a key, and then disconnect to reduce active connections

.. code:: python

    x = tr.slave.get('some key')
    tr.slave.disconnect()

and afterward reconnection will happen seamlessly as needed \o/
and chances are you'll hit a different slave

.. code:: python

    x = tr.slave.get('some other key')

or if you want to disconnect both ``tr.master`` and ``tr.slave``,
``tr.disconnect()`` can be used. it calls ``disconnect()`` on both
the slave and master clients:

.. code:: python

    x = tr.slave.get('some key')
    tr.master.publish('topic', x)
    tr.disconnect()

    # ... and later on reconnect seamlessly
    tr.master.set('some key', 'le totes!')
    x = tr.slave.get('some_key')


~~~~~~~
install
~~~~~~~
``pip install twiceredis`` or clone the repo and ``python setup.py install``
