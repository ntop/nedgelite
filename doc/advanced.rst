Advanced Features
=================

This section covers advanced nEdge Lite features and optimization techniques for experienced users.

Multi-Queue Processing
-----------------------

nEdge Lite supports distributing packet processing across multiple CPU cores using multiple NFQUEUE instances.

Configuring Multiple Queues
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**iptables Configuration:**

.. code-block:: console

   # Configure 4 queues with CPU fanout
   sudo iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark
   sudo iptables -t mangle -A PREROUTING -m mark --mark 2 -j DROP
   sudo iptables -t mangle -A PREROUTING -m mark --mark 0 \
     -j NFQUEUE --queue-balance 0:3 --queue-cpu-fanout
   sudo iptables -t mangle -A POSTROUTING -j CONNMARK --save-mark

**Start nEdge Lite:**

.. code-block:: console

   sudo nedgelite -q 0:4 -z tcp://127.0.0.1:1234

This creates 4 threads, each handling one queue (0, 1, 2, 3).

Performance Benefits
~~~~~~~~~~~~~~~~~~~~

- **Parallel Processing**: Distributes packet inspection across CPU cores
- **Higher Throughput**: Scales linearly with number of queues (up to available cores)
- **Lower Latency**: Reduces per-packet processing time

Optimal Queue Count
~~~~~~~~~~~~~~~~~~~

**Recommended**:

.. code-block:: console

   # Use number of CPU cores
   CORES=$(nproc)
   sudo nedgelite -q 0:$CORES -z tcp://127.0.0.1:1234

**Considerations**:

- More queues = more threads = higher CPU usage
- Diminishing returns beyond physical core count
- Consider hyperthreading and other workloads

CPU Affinity
~~~~~~~~~~~~

Pin nEdge Lite threads to specific CPU cores for better cache locality:

.. code-block:: console

   # Pin to cores 0-3
   sudo taskset -c 0-3 nedgelite -q 0:4 -z tcp://127.0.0.1:1234

Performance Optimization
------------------------

System Tuning
~~~~~~~~~~~~~

**Increase Conntrack Table Size:**

.. code-block:: console

   # For high connection counts
   sudo sysctl -w net.netfilter.nf_conntrack_max=1048576

   # Make persistent
   echo "net.netfilter.nf_conntrack_max=1048576" | \
     sudo tee -a /etc/sysctl.conf

**Adjust Conntrack Timeouts:**

.. code-block:: console

   # Reduce TCP established timeout
   sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=3600

   # Reduce TCP close-wait timeout
   sudo sysctl -w net.netfilter.nf_conntrack_tcp_timeout_close_wait=30

**Increase NFQUEUE Buffer:**

.. code-block:: console

   # In iptables rule
   sudo iptables -t mangle -A PREROUTING -m mark --mark 0 \
     -j NFQUEUE --queue-num 0 --queue-bypass --queue-size 4096

Memory Management
~~~~~~~~~~~~~~~~~

**Monitor Memory Usage:**

.. code-block:: console

   # Check nEdge Lite memory usage
   ps aux | grep nedgelite
   pmap $(pidof nedgelite)

**Flow Hash Size**: The flow hash table is fixed at compile time. For very high flow counts, consider increasing the hash table size in the source code.

IPv6 Support
------------

nEdge Lite supports IPv6 packet inspection alongside IPv4.

Enabling IPv6
~~~~~~~~~~~~~

The sample scripts described in the Quick Start Guide already support IPv6, it can be enabled by adding the -6 parameter.

**Load IPv6 Conntrack Module:**

.. code-block:: console

   sudo modprobe nf_conntrack_ipv6

**Configure ip6tables:**

.. code-block:: console

   sudo ip6tables -t mangle -A PREROUTING -j CONNMARK --restore-mark
   sudo ip6tables -t mangle -A PREROUTING -m mark --mark 2 -j DROP
   sudo ip6tables -t mangle -A PREROUTING -m mark --mark 0 -j NFQUEUE --queue-num 0
   sudo ip6tables -t mangle -A POSTROUTING -j CONNMARK --save-mark

**Start nEdge Lite:**

No special options needed. nEdge Lite automatically handles both IPv4 and IPv6.

.. code-block:: console

   sudo nedgelite -q 0 -z tcp://127.0.0.1:1234

IPv6 Considerations
~~~~~~~~~~~~~~~~~~~

- **Flow Export**: IPv6 flows are exported to ntopng like IPv4
- **Policy Rules**: IP pools can include IPv6 CIDR ranges

Custom nDPI Configuration
--------------------------

nDPI supports custom protocol definitions through the nDPI API. Refer to the nDPI documentation for details.

Detection Sensitivity
~~~~~~~~~~~~~~~~~~~~~

Some protocols have configurable detection thresholds. These are typically set in nDPI's protocol-specific detection functions.

Traffic Shaping Integration
----------------------------

While nEdge Lite itself doesn't perform traffic shaping, it can integrate with Linux tc (traffic control).

Using CONNMARK for tc
~~~~~~~~~~~~~~~~~~~~~

nEdge Lite's CONNMARK values can be used by tc to apply QoS policies:

.. code-block:: console

   # Create tc classes
   sudo tc qdisc add dev eth0 root handle 1: htb default 30
   sudo tc class add dev eth0 parent 1: classid 1:1 htb rate 100mbit
   sudo tc class add dev eth0 parent 1:1 classid 1:10 htb rate 80mbit
   sudo tc class add dev eth0 parent 1:1 classid 1:20 htb rate 20mbit

   # Use connmark to classify traffic
   sudo tc filter add dev eth0 parent 1: protocol ip prio 1 \
     handle 1 fw classid 1:10  # High priority (mark=1)

   sudo tc filter add dev eth0 parent 1: protocol ip prio 2 \
     handle 2 fw classid 1:20  # Low priority (mark=2)

.. _connection-labels:

Connection Labels
-----------------

In addition to the pass/drop mark, nEdge Lite can tag each connection with conntrack
labels (connlabels) describing what nDPI detected: protocols, categories and flow risks.
Labels are stored in the conntrack entry, so any iptables rule can match on them with the
``connlabel`` match, for example to drop, log or mark traffic of a given application
independently of the nEdge Lite policies.

The kernel provides 128 label bits per connection, while nDPI detects several hundred
protocols. A label map file selects the protocols, categories and risks of interest and
assigns each of them a label bit.

Label Map File
~~~~~~~~~~~~~~

The label map is a text file with one mapping per line; ``#`` starts a comment:

.. code-block:: text

   <bit> protocol <name|id>                     [<label>]
   <bit> category <name|id>                     [<label>]
   <bit> risk     <NDPI_xxx code|short name|id> [<label>]
   <bit> over                                   [<label>]

- ``<bit>``: label bit, 0-127.
- ``protocol``: nDPI protocol name or id (see ``nedgelite -H``). Both the master and the
  application protocol are matched, so ``TLS`` matches TLS.Google too. ``Unknown``
  matches flows nDPI could not detect.
- ``category``: nDPI category name or id (e.g. ``SocialNetwork``, ``Streaming``).
- ``risk``: nDPI flow risk, as ``NDPI_xxx`` code (e.g. ``NDPI_TLS_SELFSIGNED_CERTIFICATE``),
  short name (e.g. ``non_standard_port``) or id.
- ``over``: set on every flow once detection is completed.
- ``<label>``: optional name of the bit in ``connlabel.conf``. By default it is generated
  from the first mapping of the bit: ``NDPI-<PROTOCOL>``, ``NDPI-CAT-<CATEGORY>``,
  ``NDPI-RISK-<RISK>`` or ``NDPI-OVER``.

Names are case-insensitive. Several mappings can share the same bit: the bit is set when
any of them matches. Flows matching no mapping are not labelled (unless ``over`` is used).
Any error in the file (unknown name, bit out of range, malformed line) is reported with
its line number and nEdge Lite does not start.

Example (``/etc/nedgelite/labels.conf``):

.. code-block:: text

   0   over

   # Protocols
   1   protocol  TLS
   2   protocol  QUIC
   3   protocol  BitTorrent
   4   protocol  Facebook   FB
   5   protocol  Instagram
   6   protocol  TikTok

   # Categories
   10  category  SocialNetwork SOCIAL
   11  category  Streaming

   # Flow risks
   20  risk      NDPI_TLS_SELFSIGNED_CERTIFICATE
   21  risk      non_standard_port

Enabling Labels
~~~~~~~~~~~~~~~

1. Generate ``connlabel.conf`` from the map and install it, so that iptables and conntrack
   can refer to labels by name. Repeat this step whenever the map changes.

   .. code-block:: console

      nedgelite -L /etc/nedgelite/labels.conf -o /tmp
      sudo cp /tmp/connlabel.conf /etc/xtables/connlabel.conf

   With the example map above the generated file is:

   .. code-block:: text

      0	NDPI-OVER
      1	NDPI-TLS
      2	NDPI-QUIC
      3	NDPI-BITTORRENT
      4	FB
      10	SOCIAL
      11	NDPI-CAT-STREAMING
      20	NDPI-RISK-TLS_SELFSIGNED_CERT
      21	NDPI-RISK-NON_STANDARD_PORT

2. Add the iptables rules that use the labels. The kernel only reserves label space for
   connections created while at least one ``connlabel`` rule is loaded, so load the
   rules before starting nEdge Lite:

   .. code-block:: console

      # Drop BitTorrent and social network traffic
      sudo iptables -A FORWARD -m connlabel --label NDPI-BITTORRENT -j DROP
      sudo iptables -A FORWARD -m connlabel --label SOCIAL -j DROP

      # Log connections with a self-signed TLS certificate
      sudo iptables -A FORWARD -m connlabel --label NDPI-RISK-TLS_SELFSIGNED_CERT \
        -j LOG --log-prefix "self-signed: "

3. Start nEdge Lite with ``-L`` (or ``--labels=/etc/nedgelite/labels.conf`` in the
   configuration file):

   .. code-block:: console

      sudo nedgelite -q 0 -r /etc/nedgelite/policy.json -L /etc/nedgelite/labels.conf

4. Check the labels assigned to connections:

   .. code-block:: console

      sudo conntrack -L -o labels

Notes
~~~~~

- Labels are set once per connection, when nDPI detection completes, together with the
  pass/drop mark. The packets exchanged before that point are not affected by label rules.
- Labels are independent of the nEdge Lite policies: a connection dropped by a policy is
  labelled as well, and label rules apply on top of the policy verdict.
- To enforce traffic with label rules only, simply do not configure drop policies: without
  ``-r`` and ``-p`` nEdge Lite uses a built-in pass-all policy. Connections are still
  marked as passed once detection completes, which is required to stop sending them to
  nEdge Lite, so keep the mark rules installed by the setup scripts.
- Only TCP and UDP connections are labelled; broadcast and multicast traffic is skipped.
- ``nedgelite -O <dir>`` generates ``protocols.conf``, with one
  ``NDPI-<NAME>,<protocol>,<category>`` line per nDPI protocol, which can be used to look up
  the protocol and category of the generated protocol label names.

Bridge Mode Advanced Configuration
-----------------------------------

VLAN Support
~~~~~~~~~~~~

nEdge Lite works with VLAN-tagged traffic in bridge mode:

.. code-block:: console

   # Create bridge
   sudo ip link add name br0 type bridge
   sudo ip link set dev br0 up

   # Add VLAN interfaces
   sudo ip link add link eth0 name eth0.10 type vlan id 10
   sudo ip link add link eth0 name eth0.20 type vlan id 20
   sudo ip link set dev eth0.10 master br0
   sudo ip link set dev eth0.20 master br0
   sudo ip link set dev eth0.10 up
   sudo ip link set dev eth0.20 up

**Configure iptables for bridge:**

.. code-block:: console

   sudo iptables -t mangle -A PREROUTING -i br0 -j CONNMARK --restore-mark
   sudo iptables -t mangle -A PREROUTING -i br0 -m mark --mark 0 \
     -j NFQUEUE --queue-num 0
   sudo iptables -t mangle -A POSTROUTING -o br0 -j CONNMARK --save-mark

Debugging and Development
--------------------------

Verbose Logging
~~~~~~~~~~~~~~~

Enable detailed logging:

.. code-block:: console

   sudo nedgelite -q 0 -z tcp://127.0.0.1:1234 -v

Core Dumps
~~~~~~~~~~

Enable core dumps for debugging crashes:

.. code-block:: console

   ulimit -c unlimited
   sudo sysctl -w kernel.core_pattern=/tmp/core-%e-%p-%t

Running Under Debugger
~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   sudo gdb --args nedgelite -q 0 -z tcp://127.0.0.1:1234 -v

Packet Capture Integration
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Capture packets for offline analysis:

.. code-block:: console

   # Capture packets going to NFQUEUE
   sudo tcpdump -i eth0 -w /tmp/capture.pcap

   # Analyze with Wireshark or tcpdump
   tcpdump -r /tmp/capture.pcap -n

Best Practices for Production
------------------------------

1. **Use Multiple Queues**: Distribute load across CPU cores
2. **Monitor Performance**: Watch CPU, memory, and queue depth
3. **Tune Update Interval**: Balance real-time visibility with performance
4. **Enable Queue Bypass**: Prevent packet loss if nEdge Lite crashes
5. **Regular Maintenance**: Update nDPI for new protocols
6. **Backup Policies**: Keep policy files under version control
7. **Test Changes**: Verify policy changes in a test environment first
8. **Monitor Dropped Flows**: Ensure legitimate traffic isn't blocked
9. **Plan Capacity**: Size hardware for peak traffic loads

Next Steps
----------

- Review :doc:`troubleshooting` for common issues
- Learn about :doc:`architecture` for technical details
- Explore :doc:`configuration` for all available options
