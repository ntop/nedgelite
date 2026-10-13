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
- ``category``: nDPI category name or id (e.g. ``SocialNetwork``, ``Streaming``; see
  ``nedgelite -H``). Names containing spaces (``User custom category N``) must be given by id.
- ``risk``: nDPI flow risk, as ``NDPI_xxx`` code (e.g. ``NDPI_TLS_SELFSIGNED_CERTIFICATE``),
  short name (e.g. ``non_standard_port``) or id (see ``nedgelite -H``).
- ``over``: set on every flow once detection is completed.
- ``<label>``: optional name of the bit in ``connlabel.conf``. By default it is generated
  from the first mapping of the bit: ``NDPI-<PROTOCOL>``, ``NDPI-CAT-<CATEGORY>``,
  ``NDPI-RISK-<RISK>`` or ``NDPI-OVER`` (uppercase, other characters replaced by ``-``).
  Labels may only contain letters, digits and ``-`` (a ``connlabel.conf`` restriction).

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

1. Generate ``connlabel.conf`` from the map and install it as
   ``/etc/xtables/connlabel.conf``, so that iptables and conntrack can refer to labels by
   name (the ``/etc/xtables`` directory usually does not exist and must be created).
   Repeat this step whenever the map changes.

   .. code-block:: console

      nedgelite -L /etc/nedgelite/labels.conf -o /tmp
      sudo mkdir -p /etc/xtables
      sudo cp /tmp/connlabel.conf /etc/xtables/connlabel.conf

   With the example map above the generated file is:

   .. code-block:: text

      0	NDPI-OVER
      1	NDPI-TLS
      2	NDPI-QUIC
      3	NDPI-BITTORRENT
      4	FB
      5	NDPI-INSTAGRAM
      6	NDPI-TIKTOK
      10	SOCIAL
      11	NDPI-CAT-STREAMING
      20	NDPI-RISK-TLS-SELFSIGNED-CERT
      21	NDPI-RISK-NON-STANDARD-PORT

   .. note::

      nftables reads labels from ``/etc/connlabel.conf`` instead. When using ``ct label``
      in nft rules, install the same file there too (e.g. as a symlink to
      ``/etc/xtables/connlabel.conf``).

2. Load at least one ``connlabel`` rule. The kernel reserves label space only for
   connections created while a ``connlabel`` rule is loaded: without one, labels set by
   nEdge Lite are silently discarded. The rules must be added **after** the netfilter setup
   scripts (``bridge_setup.sh`` and ``default_setup.sh`` flush the iptables tables).

   Any rule using the ``connlabel`` match is enough, including a match-only rule without a
   target (the packet continues unchanged):

   .. code-block:: console

      sudo modprobe xt_connlabel
      sudo iptables -t mangle -I PREROUTING -m connlabel --label NDPI-OVER

   Rules that act on labels can be added as needed, for example:

   .. code-block:: console

      # Drop BitTorrent and social network traffic
      sudo iptables -A FORWARD -m connlabel --label NDPI-BITTORRENT -j DROP
      sudo iptables -A FORWARD -m connlabel --label SOCIAL -j DROP

      # Log connections with a self-signed TLS certificate
      sudo iptables -A FORWARD -m connlabel --label NDPI-RISK-TLS-SELFSIGNED-CERT \
        -j LOG --log-prefix "self-signed: "

   Connections that already existed when the first ``connlabel`` rule was loaded cannot be
   labelled. Flush the conntrack table to have them recreated (this also resets their
   pass/drop mark, so they are inspected again):

   .. code-block:: console

      sudo conntrack -F

3. Start nEdge Lite with ``-L`` (or ``--labels=/etc/nedgelite/labels.conf`` in the
   configuration file):

   .. code-block:: console

      sudo nedgelite -q 0 -L /etc/nedgelite/labels.conf

   When nEdge Lite runs as a systemd service, the setup script is executed at every
   service start and flushes the label rules. Add them back from a drop-in, whose
   ``ExecStartPre`` commands run after the ones of the service (i.e. after the setup
   script) and before nEdge Lite starts:

   .. code-block:: console

      sudo systemctl edit nedgelite

   .. code-block:: ini

      [Service]
      ExecStartPre=/usr/sbin/iptables -t mangle -I PREROUTING -m connlabel --label NDPI-OVER
      ExecStartPre=/usr/sbin/iptables -A FORWARD -m connlabel --label SOCIAL -j DROP

4. Check the labels assigned to connections:

   .. code-block:: console

      sudo conntrack -L -o labels

   Labels are reported by name at the end of each entry, after the mark:

   .. code-block:: text

      tcp      6 431996 ESTABLISHED src=192.168.1.20 dst=157.240.231.35 sport=52814 dport=443 packets=42 bytes=6120 src=157.240.231.35 dst=192.168.1.20 sport=443 dport=52814 packets=55 bytes=48211 [ASSURED] mark=1 use=1 labels=NDPI-OVER,NDPI-TLS,FB,SOCIAL
      tcp      6 431990 ESTABLISHED src=192.168.1.31 dst=203.0.113.7 sport=40122 dport=8443 packets=12 bytes=1830 src=203.0.113.7 dst=192.168.1.31 sport=8443 dport=40122 packets=10 bytes=4420 [ASSURED] mark=1 use=1 labels=NDPI-OVER,NDPI-TLS,NDPI-RISK-TLS-SELFSIGNED-CERT,NDPI-RISK-NON-STANDARD-PORT

   Only bits named in ``/etc/xtables/connlabel.conf`` are shown. Connections with a given
   label can be listed with ``-l``:

   .. code-block:: console

      sudo conntrack -L -o labels -l SOCIAL

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
- Labels are written to the conntrack entry via netlink, which requires the entry to be
  already confirmed by the kernel. Flows detected on their very first packet (e.g. short
  UDP exchanges such as DNS) may therefore remain unlabelled; with ``-v`` nEdge Lite logs
  ``Unable to set flow label`` for them.

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
