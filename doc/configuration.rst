Configuration
=============

This section covers all configuration options available in nEdge Lite, including command-line arguments, environment variables, and runtime behavior.

Command-Line Options
--------------------

Basic Options
~~~~~~~~~~~~~

**-q, --queue-id <id>[:<num>]**

Specifies the NFQUEUE ID(s) to listen on.

- Single queue: ``-q 0``
- Multiple queues: ``-q 0:4`` (uses queues 0, 1, 2, 3)

The number of queues determines thread count. Each queue is handled by a separate thread.

**Examples**:

.. code-block:: console

   # Single queue
   sudo nedgelite -q 0

   # Four queues (0-3) for load balancing
   sudo nedgelite -q 0:4

   # Eight queues starting at queue 10
   sudo nedgelite -q 10:8

**-v, --verbose**

Enable verbose logging output. Shows detailed information about packet processing, protocol detection, and flow management.

.. code-block:: console

   sudo nedgelite -q 0 -v

**-h, --help**

Display help message with all available options.

.. code-block:: console

   nedgelite --help

**-V, --version**

Display version information.

.. code-block:: console

   nedgelite --version

**-H**

Display list of supported nDPI protocols.

.. code-block:: console

   nedgelite -H

Policy Options
~~~~~~~~~~~~~~

**-r, --rules <path>**

Load policy rules from a JSON file. The file contains pool definitions and policy rules.

.. code-block:: console

   sudo nedgelite -q 0 -r /etc/nedgelite/policy.json

When using this option, you can reload policies by sending SIGHUP to the nEdge Lite process:

.. code-block:: console

   sudo kill -HUP $(pidof nedgelite)

**-p, --zmq-policy-endpoint <url>**

Subscribe to policy updates from ntopng via ZeroMQ. This enables dynamic policy management through ntopng's web interface.

.. code-block:: console

   sudo nedgelite -q 0 -p tcp://127.0.0.1:5557

Multiple endpoints are supported for redundancy:

.. code-block:: console

   sudo nedgelite -q 0 -p tcp://127.0.0.1:5557 -p tcp://10.0.0.1:5557

Flow Export Options
~~~~~~~~~~~~~~~~~~~

**-z, --zmq-flow-endpoint <url>**

Send flow data to ntopng via ZeroMQ. This is typically required for monitoring and visualization.

.. code-block:: console

   sudo nedgelite -q 0 -z tcp://127.0.0.1:1234

Multiple endpoints are supported to send flows to multiple ntopng instances:

.. code-block:: console

   sudo nedgelite -q 0 -z tcp://192.168.1.10:1234 -z tcp://192.168.1.20:1234

**-j, --json**

Export flows in JSON format instead of the default TLV (Type-Length-Value) format.

.. code-block:: console

   sudo nedgelite -q 0 -z tcp://127.0.0.1:1234 -j

**Note**: TLV format is more compact and efficient. Use JSON only if required for debugging or custom integrations.

**-f, --flush**

Flush flows immediately over ZMQ without batching. This reduces latency but may increase overhead.

.. code-block:: console

   sudo nedgelite -q 0 -z tcp://127.0.0.1:1234 -f

**-u, --flow-update <seconds>**

Set the interval (in seconds) for periodic flow updates. Default is 30 seconds.

.. code-block:: console

   # Update flows every 10 seconds
   sudo nedgelite -q 0 -z tcp://127.0.0.1:1234 -u 10

**Range**: 3-120 seconds

**-y, --zmq-encryption-key <key>**

Enable ZeroMQ CURVE encryption for secure flow export. Provide the server's public key.

.. code-block:: console

   sudo nedgelite -q 0 -z tcp://127.0.0.1:1234 -y "server-public-key"

Connection Tracking Options
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**-s, --skip-ct-reset**

Skip conntrack initialization at startup. By default, nEdge Lite:

- Enables conntrack accounting
- Enables conntrack timestamps
- Flushes nfacct counters

Use this option if these settings are already configured or managed externally.

.. code-block:: console

   sudo nedgelite -q 0 -s

Connection Label Options
~~~~~~~~~~~~~~~~~~~~~~~~

nEdge Lite can tag connections with conntrack labels (connlabels) describing what nDPI
detected, so that iptables rules can match on them. See :ref:`connection-labels` for
details and examples.

**-L, --labels <path>**

Enable connection labels, using the specified label map file. The map selects which nDPI
protocols, categories and flow risks set which of the 128 available label bits.

.. code-block:: console

   sudo nedgelite -q 0 -L /etc/nedgelite/labels.conf

**-o, --connlabel <dir>**

Generate ``<dir>/connlabel.conf`` from the label map, then exit. The file assigns a name
to each label bit used by the map and must be installed as ``/etc/xtables/connlabel.conf``
so that iptables and conntrack can refer to labels by name. Requires ``-L``.

.. code-block:: console

   nedgelite -L /etc/nedgelite/labels.conf -o /tmp
   sudo cp /tmp/connlabel.conf /etc/xtables/connlabel.conf

**-O, --protocols <dir>**

Generate ``<dir>/protocols.conf`` with one ``NDPI-<NAME>,<protocol>,<category>`` line per
nDPI protocol, then exit. It can be combined with ``-o``.

.. code-block:: console

   nedgelite -O /tmp

License Options
~~~~~~~~~~~~~~~

**--show-system-id**

Display the system ID used for license generation.

.. code-block:: console

   nedgelite --show-system-id

**--check-license**

Check the validity of the installed license.

.. code-block:: console

   nedgelite --check-license

**--check-maintenance**

Check the maintenance expiration date.

.. code-block:: console

   nedgelite --check-maintenance

Configuration Files
-------------------

License File
~~~~~~~~~~~~

**Locations** (checked in order):

1. ``nedgelite.license`` (current directory)
2. ``/etc/nedgelite.license``

**Format**: Binary license file provided by ntop.org

Policy File
~~~~~~~~~~~

**Location**: Specified with ``-r`` option

**Format**: JSON (newline-delimited JSON objects)

**Example**: ``/etc/nedgelite/policy.json``

See :doc:`policies` for detailed policy file format.

Label Map File
~~~~~~~~~~~~~~

**Location**: Specified with ``-L`` option

**Format**: Text, one ``<bit> <type> <name> [<label>]`` mapping per line

**Example**: ``/etc/nedgelite/labels.conf``

See :ref:`connection-labels` for the file format.

Runtime Configuration
---------------------

Signals
~~~~~~~

nEdge Lite responds to POSIX signals:

**SIGINT / SIGTERM**

Gracefully shutdown nEdge Lite:

.. code-block:: console

   sudo kill -TERM $(pidof nedgelite)
   # or press Ctrl+C

**SIGHUP**

Reload policy rules from file (only if ``-r`` option was used):

.. code-block:: console

   sudo kill -HUP $(pidof nedgelite)

System Settings
~~~~~~~~~~~~~~~

nEdge Lite requires specific kernel settings for optimal operation:

**Conntrack Accounting**:

.. code-block:: console

   echo 1 | sudo tee /proc/sys/net/netfilter/nf_conntrack_acct

**Conntrack Timestamps**:

.. code-block:: console

   echo 1 | sudo tee /proc/sys/net/netfilter/nf_conntrack_timestamp

**Conntrack Table Size** (for high connection counts):

.. code-block:: console

   # Increase conntrack table size
   echo 262144 | sudo tee /proc/sys/net/netfilter/nf_conntrack_max

**Conntrack Timeouts** (adjust as needed):

.. code-block:: console

   # TCP established timeout (default: 5 days)
   echo 3600 | sudo tee /proc/sys/net/netfilter/nf_conntrack_tcp_timeout_established

Make these settings persistent by adding them to ``/etc/sysctl.conf``:

.. code-block:: console

   net.netfilter.nf_conntrack_acct=1
   net.netfilter.nf_conntrack_timestamp=1
   net.netfilter.nf_conntrack_max=262144
   net.netfilter.nf_conntrack_tcp_timeout_established=3600

Common Configuration Examples
------------------------------

Standalone with Static Policy
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   sudo nedgelite -q 0 -r /etc/nedgelite/policy.json -v

Integrated with ntopng
~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   # On ntopng host
   sudo ntopng -i tcp://0.0.0.0:5556c --zmq-publish-events tcp://0.0.0.0:5557

   # On nEdge Lite host
   sudo nedgelite -q 0 -z tcp://ntopng-server:5556 -p tcp://ntopng-server:5557

Multi-Queue for Performance
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   # Configure 8 queues
   sudo nedgelite -q 0:8 -z tcp://127.0.0.1:1234 -u 15

Bridge Mode with Policy File
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   # Set up bridge
   sudo /usr/share/nedgelite/scripts/bridge_setup.sh eth0 eth1

   # Start nEdge Lite
   sudo nedgelite -q 0 -r /etc/nedgelite/policy.json -z tcp://127.0.0.1:1234

Connection Labels
~~~~~~~~~~~~~~~~~

.. code-block:: console

   # Generate and install connlabel.conf (once, and after changing the map)
   nedgelite -L /etc/nedgelite/labels.conf -o /tmp
   sudo cp /tmp/connlabel.conf /etc/xtables/connlabel.conf

   # Start nEdge Lite with labels enabled
   sudo nedgelite -q 0 -r /etc/nedgelite/policy.json -L /etc/nedgelite/labels.conf

Multiple ntopng Collectors
~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   sudo nedgelite -q 0 \
     -z tcp://ntopng1.local:5556 \
     -z tcp://ntopng2.local:5556 \
     -p tcp://ntopng1.local:5557

Encrypted Flow Export
~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   sudo nedgelite -q 0 \
     -z tcp://remote-ntopng:5556 \
     -y "Yne@$w-vo<fVvi]a<NY6T1ed:M$fCG*[IaLV{hID" \
     -p tcp://remote-ntopng:5557

Performance Tuning
------------------

For high-traffic environments:

**Multi-Queue Setup**:

.. code-block:: console

   # Use one queue per CPU core
   sudo nedgelite -q 0:$(nproc) -z tcp://127.0.0.1:1234 -u 10

**Optimize Flow Updates**:

.. code-block:: console

   # Reduce update interval for real-time monitoring
   sudo nedgelite -q 0 -z tcp://127.0.0.1:1234 -u 5

   # Increase update interval for high-volume environments
   sudo nedgelite -q 0 -z tcp://127.0.0.1:1234 -u 60

**CPU Affinity**:

.. code-block:: console

   # Pin to specific CPU cores
   sudo taskset -c 0-3 nedgelite -q 0:4 -z tcp://127.0.0.1:1234

Next Steps
----------

- Learn how to write :doc:`policies` for traffic filtering
- Set up :doc:`ntopng_integration` for centralized management
- Explore :doc:`advanced` features and optimizations
