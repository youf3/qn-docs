Link Emulation & Probing
========================

QNCP includes built-in link connection emulation and dual-plane probing so
multi-node control-plane workflows, link state transitions, and hardware
failure modes can be tested over standard IPv4/IPv6 networks without
physical optical hardware.

Overview & Dual-Plane Architecture
----------------------------------

Link emulation separates signaling from emulated photon transport across two
planes:

- **Control Plane (MQTT)**: ``LinkAdjacencyManager`` and ``LinkInterpreter``
  exchange ``link.hello`` and ``link.probe`` RPC messages on per-agent topics
  (``rpc/link/<agent-id>``) and emit ``linkStateUpdate`` telemetry to the
  ``monitor`` topic.
- **Emulated Optical Data Plane (UDP)**: On quantum channels, ``DummyLightSrc``
  transmits non-blocking IPv4/IPv6 UDP datagrams carrying optical metadata
  and probe identifiers to ``DummySignalMeasurement``, which acts as an
  emulated single-photon detector and time-tagger.

::

    Agent A (TX)                                         Agent B (RX)
    ┌──────────────────────┐    MQTT: link.hello /       ┌──────────────────────┐
    │ LinkAdjacencyManager │◄── link.probe (control) ───►│   LinkInterpreter    │
    └──────────┬───────────┘                             └──────────▲───────────┘
               │ generate()                                         │ measure()
    ┌──────────▼───────────┐    UDP: JSON probe packet   ┌──────────┴───────────┐
    │    DummyLightSrc     │════════════════════════════►│DummySignalMeasurement│
    └──────────────────────┘    port = base + crc32(id)  └──────────────────────┘

Link Connection Emulation
-------------------------

Both ``DummyLightSrc`` and ``DummySignalMeasurement`` (defined in
``quantnet_agent.hal.driver.dummy``) construct directional virtual links
(``VirtualLinkEndpoint``) directly from each node's topology definition JSON
(``conf_*.json``) using ``parse_virtual_links()``.

Channel Parsing & Virtual Link IDs
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``parse_virtual_links(node_file, ip="127.0.0.1", base_port=15000, port_range=10000, peer_ips=None)``
reads channel entries from both ``channels`` and
``matterLightInterfaceSettings[].channels`` in ``node_file``. For every
channel containing a ``ID``, ``direction`` (``"out"`` or ``"in"``), and a
``neighbor`` object (``systemRef`` or ``idRef``, plus ``channelRef``), it
constructs a directional ``VirtualLinkEndpoint``:

- **Outbound (``direction: "out"``)**: Indexed under
  ``(dst_cid, local_ch)`` with canonical virtual link ID:

  .. code-block:: text

      {local_cid}:{local_ch}->{neighbor_cid}:{neighbor_ch}

- **Inbound (``direction: "in"``)**: Indexed under
  ``(src_cid, src_ch)`` with canonical virtual link ID:

  .. code-block:: text

      {neighbor_cid}:{neighbor_ch}->{local_cid}:{local_ch}

Channel ``type`` defaults to ``"quantum"`` if omitted, and optical attributes
(``wavelength`` defaulting to ``1550.0`` nm, ``power`` defaulting to ``0.0``
dBm) are extracted automatically for TX emission and RX reporting.

Deterministic CRC32 Port Derivation
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

TX and RX endpoints derive the exact same UDP port independently from
topology metadata without out-of-band port negotiation:

.. code-block:: python

    port = base_port + (zlib.crc32(vlink_id.encode("utf-8")) % port_range)

With default settings (``base_port = 15000``, ``port_range = 10000``), all
virtual links map deterministically into UDP ports ``15000``–``24999``.
``DummySignalMeasurement`` binds a non-blocking ``SOCK_DGRAM`` socket (with
``SO_REUSEADDR`` and ``SO_REUSEPORT`` when available) on ``(local_ip, port)``
for every inbound virtual link at agent startup.

Single-Host vs. Multi-Host Overlay Addressing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The driver ``ip`` and ``peer_ips`` options support three deployment modes,
automatically choosing ``AF_INET`` (IPv4) or ``AF_INET6`` (IPv6) sockets:

- **Single-host loopback**: Leave ``ip`` at its default (``127.0.0.1``, or
  ``::1``) and omit ``peer_ips``. All agents transmit and listen on loopback.
- **Docker Compose bridge/host**: Set ``ip`` to the container/host bind
  address and supply ``peer_ips`` mapping neighbor CIDs to their reachable
  container or host IPs.
- **Multi-site WAN / FABRIC (e.g., FABNetv6)**: Set ``ip`` to the VM's local
  data-plane IPv6 address and configure ``peer_ips`` as a comma-separated
  ``CID:IP`` or ``CID=IP`` mapping (or a dict) so UDP probes traverse the
  dedicated experiment network (such as FABRIC ``FABNetv6``) separately from
  the management plane.

Classical Link Probing Emulation
--------------------------------

Classical channels (``type != "quantum"``, ``is_quantum = False``) validate
control-plane reachability continuously without optical hardware probes:

1. **Adjacency Establishment**: ``LinkAdjacencyManager`` sends periodic
   ``link.hello`` MQTT RPCs every ``hello_interval`` seconds.

   - **Bilateral confirmation**: When a neighbor's ``seen_neighbors`` list
     includes ``<local_cid>`` or ``<local_cid>:<ch>``, outbound ``INIT``
     entries promote to ``CONTROL_UP``.
   - **Unidirectional fallback**: When the local agent has no inbound channel
     configured from that neighbor, receiving ``linkHelloResponse(status="ok")``
     immediately transitions the outbound link from ``INIT`` to ``CONTROL_UP``.

2. **State Ceiling**: Classical links stay capped at ``CONTROL_UP``;
   ``_probe_if_ready()`` and ``_send_probe()`` skip optical ``link.probe``
   requests for non-quantum channels.
3. **Continuous RTT Probing**: Once a classical link is in ``CONTROL_UP``,
   each ``link.hello`` RPC measures round-trip time (``rtt_ms``) and records
   a ``kind="hello"`` sample via ``LinkStateTable.record_probe()``
   (``probe_id``, ``status="ok"`` or ``"rpc_error"``, ``rtt_ms``, ``rx_count``,
   ``vlink``). The receiving ``LinkInterpreter.handle_hello()`` increments its
   inbound entry's ``probe_rx_count`` and ``last_probe_received``, while the
   sender emits periodic ``linkStateUpdate`` telemetry (``state_changed=False``).

Quantum Link Probing Emulation
------------------------------

Quantum channels (``type == "quantum"``, ``is_quantum = True``) pair UDP
optical probe emission with MQTT RPC detector gating before promoting a
link to ``QUANTUM_UP``.

Pre-Flight Readiness & Switch Gating
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Before emitting a quantum probe, ``LinkAdjacencyManager._probe_if_ready()``
verifies three preconditions:

1. **Local calibration readiness**: Local ``node.status`` must be
   ``IN_SPEC`` (all local calibration tasks passing). If local readiness
   drops while links are ``QUANTUM_UP``, those links demote to ``CONTROL_UP``.
2. **Remote calibration readiness**: The neighbor's ``ready`` boolean
   (exchanged in ``link.hello`` and ``linkHelloResponse`` and re-verified by
   ``LinkInterpreter.handle_probe()``) must be ``True`` (``IN_SPEC``). If
   the neighbor reports ``status="not_ready"``, probing is skipped (or demotes
   ``QUANTUM_UP`` to ``CONTROL_UP``).
3. **Optical switch routing (when applicable)**: If ``switch_in_path`` is
   ``True``, the agent calls ``link.switchPortCheck`` on ``switch_cid`` and
   proceeds only when status is ``"ok"``.

Two-Plane Optical Probe Sequence
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When all preconditions pass on an outbound quantum link in ``CONTROL_UP`` or
``QUANTUM_UP``:

1. **UDP Emission (``DummyLightSrc.generate``)**:
   Applies configured ``failrate``; on pass, emits a JSON UDP datagram to
   ``(vlink.peer_ip, vlink.port)`` containing ``vlink``, ``probe_id``,
   ``src_cid``, ``dst_cid``, ``src_ch``, ``dst_ch``, ``wavelength``,
   ``power``, ``pol`` (``"H"``), and emission epoch timestamp ``tx_ts``.
   If UDP emission fails (``rc != 0``), the probe is recorded as
   ``tx_failed`` and any ``QUANTUM_UP`` link demotes to ``CONTROL_UP``.
2. **MQTT Detector Gate (``link.probe``)**:
   ``LinkAdjacencyManager`` sends ``link.probe`` (``src_cid``, ``dst_cid``,
   ``channel_id``, ``probe_id``) to ``rpc/link/<dst_cid>``.
3. **Time-Tagger Coincidence (``DummySignalMeasurement.measure``)**:
   ``LinkInterpreter.handle_probe()`` locates the matching inbound channel
   entry (rejecting probes addressed to non-inbound channels) and invokes
   ``measure()``. The driver drains its UDP socket into a 256-entry ring
   buffer, discards tags older than the coincidence ``window`` (default
   ``2.0`` s), matches ``vlink``, ``src_cid``, ``src_ch``, ``dst_cid``, and
   ``probe_id``, applies detector ``efficiency`` (default ``1.0``), and
   computes one-way Time-of-Flight ``tof = max(0.0, rx_ts - tx_ts)``.
4. **State Transition & Telemetry**:

   - On ``status == "ok"``: promotes ``CONTROL_UP`` → ``QUANTUM_UP`` (or
     keeps ``QUANTUM_UP``) and records ``detected``, ``counts``, ``tof``,
     ``rx_count``, and ``vlink`` in ``probe_history``.
   - On failure (``timeout``, ``detector_inefficiency``,
     ``invalid_inbound_channel``, ``rpc_error``): demotes ``QUANTUM_UP`` →
     ``CONTROL_UP`` while leaving classical adjacency intact until
     ``hold_time`` expires.

.. note::

   Because one-way ``tof = max(0.0, rx_ts - tx_ts)`` compares timestamps across
   sender and receiver hosts, multi-VM deployments (such as FABRIC testbeds)
   should synchronize host clocks (for example with LinuxPTP ``ptp_kvm`` and
   ``phc2sys``) for accurate sub-millisecond Time-of-Flight readings.

Configuration, Fault Injection & Testing
----------------------------------------

Agent Device Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~

Configure ``DummyLightSrc`` and ``DummySignalMeasurement`` under
``[devices]`` in ``agent.cfg``:

.. code-block:: ini

    [devices]
    # Single-host loopback example
    [[lightsource]]
    enabled = true
    type = Light_Source
    driver = DummyLightSrc
    ip = 127.0.0.1
    base_port = 15000
    port_range = 10000
    failrate = 0.0

    [[signalmeasurement]]
    enabled = true
    type = Signal_Measurement
    driver = DummySignalMeasurement
    ip = 127.0.0.1
    base_port = 15000
    port_range = 10000
    window = 2.0
    resolution = 1e-7
    efficiency = 1.0

For distributed IPv6 hosts (e.g., FABRIC ``FABNetv6``), bind each node's
data-plane address and supply ``peer_ips``:

.. code-block:: ini

    [[lightsource]]
    enabled = true
    type = Light_Source
    driver = DummyLightSrc
    ip = 2602:fcfb:1a:4::2
    base_port = 15000
    peer_ips = "QPU-1:2602:fcfb:1a:4::2,BSM-1_2:2602:fcfb:e:226::2"

    [[signalmeasurement]]
    enabled = true
    type = Signal_Measurement
    driver = DummySignalMeasurement
    ip = 2602:fcfb:1a:4::2
    base_port = 15000
    window = 2.0
    resolution = 1e-7
    peer_ips = "QPU-1:2602:fcfb:1a:4::2,BSM-1_2:2602:fcfb:e:226::2"

Driver Parameter Reference
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. list-table::
   :header-rows: 1
   :widths: 18 22 15 45

   * - Parameter
     - Driver
     - Default
     - Description
   * - ``ip``
     - Both
     - ``127.0.0.1``
     - Local IPv4 or IPv6 address to bind (RX) or fall back to (TX).
   * - ``base_port``
     - Both
     - ``15000``
     - Base UDP port added to ``crc32(vlink_id) % port_range``.
   * - ``port_range``
     - Both
     - ``10000``
     - Port modulus range (effective range ``15000``–``24999`` by default).
   * - ``peer_ips``
     - Both
     - ``None``
     - Neighbor CID-to-IP mapping (``"CID:IP,..."``, ``"CID=IP,..."``, or dict).
   * - ``failrate``
     - ``DummyLightSrc``
     - ``0``
     - Probability (``0.0``–``1.0``) that probe emission fails (``tx_failed``).
   * - ``window``
     - ``DummySignalMeasurement``
     - ``2.0``
     - Coincidence / wait timeout window in seconds (alias: ``timeout``).
   * - ``resolution``
     - ``DummySignalMeasurement``
     - ``1e-7``
     - Emulated time-tagger timing resolution in seconds.
   * - ``efficiency``
     - ``DummySignalMeasurement``
     - ``1.0``
     - Detection probability (``0.0``–``1.0``); below ``1.0`` injects ``detector_inefficiency``.

See :doc:`Link Adjacency Protocol </link-adjacency>` for the state machine,
:doc:`Remote REPL </remote-repl>` for interactive probe inspection and manual
triggers, :doc:`Monitoring </monitoring>` for controller DB persistence, and
:doc:`Northbound API </api>` for dashboard visualization.
