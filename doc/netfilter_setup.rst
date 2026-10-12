Netfilter Setup
===============

nEdge Lite relies on Linux netfilter (iptables) to intercept packets for inspection. This section explains how to configure netfilter for different deployment scenarios,
however reading this section is usually not required when using the provided scripts as explained in the Quick Start Guide section.

Understanding NFQUEUE
---------------------

NFQUEUE is a netfilter target that queues packets to userspace applications for processing. nEdge Lite uses NFQUEUE to:

1. Receive packets from the kernel
2. Perform Deep Packet Inspection
3. Apply policy decisions
4. Return verdicts (accept/drop) to the kernel

Key Concepts
~~~~~~~~~~~~

**Queue ID**: Each NFQUEUE has a numeric ID (0-65535). nEdge Lite listens on specific queue IDs specified with the ``-q`` option.

**Connection Marking (CONNMARK)**: Instead of marking individual packets, nEdge Lite marks entire connections using conntrack. This ensures all packets in a connection follow the same policy.

**Mark Values**:

- ``0``: Unmarked (needs inspection)
- ``1``: Pass (allow)
- ``2``: Drop (block)

**Packet Flow**:

.. code-block:: text

   PREROUTING → Restore CONNMARK → [mark=2? DROP] → [mark=0? NFQUEUE] → POSTROUTING → Save CONNMARK

Setup Scripts
-------------

nEdge Lite includes setup scripts for common deployment scenarios.

Single Interface Mode
~~~~~~~~~~~~~~~~~~~~~~

Use this mode when protecting traffic on a single network interface (e.g., protecting local services or gateway traffic).

**Script**: ``/usr/share/nedgelite/scripts/default_setup.sh``

**Usage**:

.. code-block:: console

   sudo /usr/share/nedgelite/scripts/default_setup.sh <interface>

**Example**:

.. code-block:: console

   sudo /usr/share/nedgelite/scripts/default_setup.sh eth0

**What it does**:

.. code-block:: bash

   # Enable conntrack accounting and timestamps
   echo 1 > /proc/sys/net/netfilter/nf_conntrack_acct
   echo 1 > /proc/sys/net/netfilter/nf_conntrack_timestamp

   # INPUT chain: local incoming traffic
   iptables -t mangle -A INPUT -j CONNMARK --restore-mark
   iptables -t mangle -A INPUT -m mark --mark 2 -j DROP
   iptables -t mangle -A INPUT -m mark --mark 0 -j NFQUEUE --queue-num 0

   # OUTPUT chain: local outgoing traffic
   iptables -t mangle -A OUTPUT -j CONNMARK --restore-mark
   iptables -t mangle -A OUTPUT -m mark --mark 2 -j DROP
   iptables -t mangle -A OUTPUT -m mark --mark 0 -j NFQUEUE --queue-num 0

   # Save marks to conntrack
   iptables -t mangle -A POSTROUTING -j CONNMARK --save-mark

Bridge Mode
~~~~~~~~~~~

Use this mode for transparent inspection between two network segments (e.g., LAN and WAN).

**Script**: ``/usr/share/nedgelite/scripts/bridge_setup.sh``

**Usage**:

.. code-block:: console

   sudo /usr/share/nedgelite/scripts/bridge_setup.sh <lan_interface> <wan_interface>

**Example**:

.. code-block:: console

   sudo /usr/share/nedgelite/scripts/bridge_setup.sh eth0 eth1

**What it does**:

.. code-block:: bash

   # Create bridge
   ip link add name br0 type bridge
   ip link set dev $LAN_IF master br0
   ip link set dev $WAN_IF master br0
   ip link set dev br0 up
   ip link set dev $LAN_IF up
   ip link set dev $WAN_IF up

   # Enable bridge netfilter
   modprobe br_netfilter
   echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables

   # Configure iptables for bridge
   iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark
   iptables -t mangle -A PREROUTING -m mark --mark 2 -j DROP
   iptables -t mangle -A PREROUTING -m mark --mark 0 -j NFQUEUE --queue-num 0
   iptables -t mangle -A POSTROUTING -j CONNMARK --save-mark

**Bridge with VLANs**:

If using VLAN trunking, you can create VLAN subinterfaces on the bridge:

.. code-block:: console

   # Add VLAN 10
   ip link add link br0 name br0.10 type vlan id 10
   ip link set dev br0.10 up
   ip addr add 192.168.10.1/24 dev br0.10

Manual Configuration
--------------------

For custom deployments, you can manually configure iptables.

Basic Manual Setup
~~~~~~~~~~~~~~~~~~

.. code-block:: console

   # Enable conntrack features
   sudo sysctl -w net.netfilter.nf_conntrack_acct=1
   sudo sysctl -w net.netfilter.nf_conntrack_timestamp=1

   # Configure iptables (example for FORWARD chain)
   sudo iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark
   sudo iptables -t mangle -A PREROUTING -m mark --mark 2 -j DROP
   sudo iptables -t mangle -A PREROUTING -m mark --mark 0 -j NFQUEUE --queue-num 0 --queue-bypass
   sudo iptables -t mangle -A POSTROUTING -j CONNMARK --save-mark

**Important Options**:

- ``--restore-mark``: Copies conntrack mark to packet mark
- ``--save-mark``: Copies packet mark back to conntrack
- ``--queue-num 0``: Specifies NFQUEUE ID (must match nEdge Lite's ``-q`` option)
- ``--queue-bypass``: If nEdge Lite is not running, packets pass through (optional, but recommended for testing)

Multiple Queue Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For load balancing across CPU cores, use multiple queues:

.. code-block:: console

   # Configure 4 queues (0-3)
   sudo iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark
   sudo iptables -t mangle -A PREROUTING -m mark --mark 2 -j DROP
   sudo iptables -t mangle -A PREROUTING -m mark --mark 0 -j NFQUEUE --queue-balance 0:3 --queue-cpu-fanout
   sudo iptables -t mangle -A POSTROUTING -j CONNMARK --save-mark

Then start nEdge Lite with:

.. code-block:: console

   sudo nedgelite -q 0:4 -z tcp://127.0.0.1:1234

**Options explained**:

- ``--queue-balance 0:3``: Distribute packets across queues 0-3
- ``--queue-cpu-fanout``: Pin queues to CPU cores for better performance

IPv6 Support
~~~~~~~~~~~~

To filter IPv6 traffic, add corresponding ip6tables rules:

.. code-block:: console

   # Enable IPv6 conntrack
   sudo modprobe nf_conntrack_ipv6

   # Configure ip6tables
   sudo ip6tables -t mangle -A PREROUTING -j CONNMARK --restore-mark
   sudo ip6tables -t mangle -A PREROUTING -m mark --mark 2 -j DROP
   sudo ip6tables -t mangle -A PREROUTING -m mark --mark 0 -j NFQUEUE --queue-num 0
   sudo ip6tables -t mangle -A POSTROUTING -j CONNMARK --save-mark

Advanced Scenarios
------------------

Router Mode
~~~~~~~~~~~

When nEdge Lite runs on a router/gateway:

.. code-block:: console

   # Enable IP forwarding
   sudo sysctl -w net.ipv4.ip_forward=1

   # Configure NAT (if needed)
   sudo iptables -t nat -A POSTROUTING -o wan0 -j MASQUERADE

   # Configure packet filtering
   sudo iptables -t mangle -A PREROUTING -j CONNMARK --restore-mark
   sudo iptables -t mangle -A PREROUTING -m mark --mark 2 -j DROP
   sudo iptables -t mangle -A PREROUTING -m mark --mark 0 -j NFQUEUE --queue-num 0
   sudo iptables -t mangle -A POSTROUTING -j CONNMARK --save-mark

Interface-Specific Filtering
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Filter only specific interfaces:

.. code-block:: console

   # Only inspect traffic on eth0
   sudo iptables -t mangle -A PREROUTING -i eth0 -j CONNMARK --restore-mark
   sudo iptables -t mangle -A PREROUTING -i eth0 -m mark --mark 2 -j DROP
   sudo iptables -t mangle -A PREROUTING -i eth0 -m mark --mark 0 -j NFQUEUE --queue-num 0
   sudo iptables -t mangle -A POSTROUTING -o eth0 -j CONNMARK --save-mark

Direction-Specific Filtering
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Inspect only incoming or outgoing traffic:

.. code-block:: console

   # Only inspect incoming traffic (from WAN)
   sudo iptables -t mangle -A PREROUTING -i wan0 -j CONNMARK --restore-mark
   sudo iptables -t mangle -A PREROUTING -i wan0 -m mark --mark 2 -j DROP
   sudo iptables -t mangle -A PREROUTING -i wan0 -m mark --mark 0 -j NFQUEUE --queue-num 0
   sudo iptables -t mangle -A POSTROUTING -j CONNMARK --save-mark

Subnet-Specific Filtering
~~~~~~~~~~~~~~~~~~~~~~~~~~

Apply filtering only to specific IP ranges:

.. code-block:: console

   # Only inspect traffic from 192.168.1.0/24
   sudo iptables -t mangle -A PREROUTING -s 192.168.1.0/24 -j CONNMARK --restore-mark
   sudo iptables -t mangle -A PREROUTING -s 192.168.1.0/24 -m mark --mark 2 -j DROP
   sudo iptables -t mangle -A PREROUTING -s 192.168.1.0/24 -m mark --mark 0 -j NFQUEUE --queue-num 0
   sudo iptables -t mangle -A POSTROUTING -j CONNMARK --save-mark

Viewing and Debugging
---------------------

View Current Rules
~~~~~~~~~~~~~~~~~~

.. code-block:: console

   # View mangle table
   sudo iptables -t mangle -L -n -v

   # View specific chain
   sudo iptables -t mangle -L PREROUTING -n -v

View NFQUEUE Statistics
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   # Check queue statistics (requires nfqueue-utils)
   cat /proc/net/netfilter/nfnetlink_queue

View Connection Tracking
~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: console

   # View all connections
   sudo conntrack -L

   # View connections with marks
   sudo conntrack -L -m

   # View connections with labels (when -L is used)
   sudo conntrack -L -o labels

   # View statistics
   sudo conntrack -S

Clearing Rules
~~~~~~~~~~~~~~

To remove all iptables rules:

.. code-block:: console

   # Flush all chains
   sudo iptables -t mangle -F

   # Delete custom chains
   sudo iptables -t mangle -X

   # Reset policies to ACCEPT
   sudo iptables -t mangle -P PREROUTING ACCEPT
   sudo iptables -t mangle -P POSTROUTING ACCEPT

Troubleshooting
---------------

Packets Not Reaching nEdge Lite
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

1. **Check iptables rules**:

   .. code-block:: console

      sudo iptables -t mangle -L -n -v

   Look for NFQUEUE rules and verify packet counters are increasing.

2. **Verify queue ID**:

   Ensure iptables ``--queue-num`` matches nEdge Lite's ``-q`` option.

3. **Check conntrack**:

   .. code-block:: console

      sudo conntrack -L | grep MARK

nEdge Lite Not Starting
~~~~~~~~~~~~~~~~~~~~~~~

1. **Check if queue is already in use**:

   .. code-block:: console

      cat /proc/net/netfilter/nfnetlink_queue

2. **Use queue-bypass**:

   Add ``--queue-bypass`` to iptables rules for testing.

Performance Issues
~~~~~~~~~~~~~~~~~~

1. **Use multiple queues**:

   Distribute load across CPU cores (see Multiple Queue Configuration above).

2. **Check CPU affinity**:

   Pin nEdge Lite threads to specific CPU cores.

3. **Monitor queue depth**:

   .. code-block:: console

      watch -n 1 'cat /proc/net/netfilter/nfnetlink_queue'

Next Steps
----------

- Learn about all :doc:`configuration` options for nEdge Lite
- Understand :doc:`policies` for traffic filtering
- Explore :doc:`advanced` features
