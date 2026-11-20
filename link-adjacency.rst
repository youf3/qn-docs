Link Adjacency Protocol
=======================

The Link Adjacency Protocol provides IS-IS-inspired neighbor discovery
and per-channel link state management for QNCP agents. Each agent actively
discovers its neighbors, establishes bilateral classical and quantum link
state, runs continuous control and optical probing, and reports state and
probe telemetry to the controller.

Overview
--------

When an agent starts, the ``LinkAdjacencyManager`` service reads the
node configuration (``channels`` and
``matterLightInterfaceSettings[].channels``) to build per-channel entries
keyed by ``<neighbor_cid>:<channel_id>`` (for example, ``BSM-1_2:1``). It
then runs a periodic Hello loop that:

1. Sends ``link.hello`` MQTT RPCs on every outbound (``direction: "out"``)
   channel, carrying local ``ready`` (``IN_SPEC``) status and
   ``seen_neighbors``.
2. Confirms bilateral adjacency (or unidirectional Hello ACK when no
   inbound return channel exists) to reach ``CONTROL_UP``.
3. Records continuous RTT telemetry (``kind="hello"``) on classical
   links, and runs two-plane UDP/MQTT quantum probes on quantum links when
   both endpoints are ``IN_SPEC``.
4. Expires outbound links whose hold timer has elapsed back to ``DOWN``.

State changes and periodic probe snapshots are published to the
``monitor`` topic as ``linkStateUpdate`` events, which the controller
persists in MongoDB and uses to filter topology edges. See
:doc:`Link Emulation & Probing </link-emulation>` for virtual link and
optical probe mechanics.

Link States
-----------

Outbound links (``direction: "out"``) transition through four states,
whereas inbound channels (``direction: "in"``) are passive receivers
(no active state machine or hold timer) and display state ``—``:

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - State
     - Description
   * - ``DOWN``
     - No communication with neighbor. Initial state, manually disconnected,
       or hold timer expired.
   * - ``INIT``
     - First Hello sent to neighbor, awaiting bilateral confirmation (or
       unidirectional Hello ACK when no inbound channel exists).
   * - ``CONTROL_UP``
     - Classical control path is confirmed. Classical links remain in
       ``CONTROL_UP``; quantum links stay here while awaiting ``IN_SPEC``
       readiness on both endpoints, switch port verification, or a passing
       optical probe.
   * - ``QUANTUM_UP``
     - Quantum probe succeeded (requires ``is_quantum=True`` and both
       endpoints ``IN_SPEC``). The optical/quantum path is verified
       end-to-end. Demotes back to ``CONTROL_UP`` if probing fails or
       either endpoint leaves ``IN_SPEC``.

State Machine
~~~~~~~~~~~~~

::

                          (both IN_SPEC + probe ok)
    DOWN ──(send hello)──► INIT ──(bilateral/ACK)──► CONTROL_UP ──────────────► QUANTUM_UP
     ▲                                                  ▲    │                     │  │
     │                                                  └────┴──(probe fail /      │  │
     │                                                           not IN_SPEC)──────┘  │
     └──────────────────────(hold timer expired / disconnect)─────────────────────────┘

Architecture
------------

The protocol is split across three components:

**LinkStateTable** (``quantnet_agent.hal.link_state_table``)
    Shared in-memory state. Stores a ``LinkEntry`` per channel keyed by
    ``<neighbor_cid>:<channel_id>``, tracking ``state``, ``direction``,
    ``is_quantum``, ``neighbor_ready``, timestamps, transition history,
    and rolling probe counters/history (``probe_tx_count``,
    ``probe_rx_count``, ``probe_ok_count``, ``probe_fail_count``,
    ``last_probe_status``, ``last_probe_info``, ``probe_history``).
    Attached to the HAL as ``hal.link_state_table``.

**LinkAdjacencyManager** (``quantnet_agent.service.link_adjacency``)
    Service-layer component that drives the outbound Hello loop, hold
    timer checks, ``IN_SPEC`` readiness propagation, and classical/quantum
    probing. Created during agent startup and attached to the HAL as
    ``hal.link_mgr``.

**LinkInterpreter** (``quantnet_agent.hal.interpreter.link``)
    Built-in command interpreter that handles inbound RPC commands:

    - ``link.hello`` — process incoming Hello, update liveness and
      remote ``ready`` state, increment classical RX probe count, and
      confirm bilateral adjacency
    - ``link.probe`` — verify ``IN_SPEC`` readiness and invoke the
      receiver's ``SignalMeasurement`` time-tagger
    - ``link.control`` — handle remote REPL control and inspection commands
      (see :doc:`Remote REPL </remote-repl>`)
    - ``link.switchPortCheck`` — (OpticalSwitch nodes only) verify
      switch port routing

Configuration
-------------

The Hello interval and hold time are configurable in ``agent.cfg``:

.. code-block:: ini

    [link]
    hello_interval = 10
    hold_time = 30

- ``hello_interval``: seconds between Hello messages (default: 10)
- ``hold_time``: seconds before an unresponsive link transitions to
  DOWN (default: 30)

These can also be changed at runtime via the REPL:

.. code-block:: text

    quantnet-agent [QPU-1]> set hello-interval 5
    quantnet-agent [QPU-1]> set hold-time 60

RPC Schema
----------

The protocol uses schemas defined in
``quantnet_mq/schema/rpc/link_adjacency.yaml``:

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Schema
     - Purpose
   * - ``linkHello`` / ``linkHelloResponse``
     - Periodic neighbor discovery messages. Payload includes
       ``src_cid``, ``dst_cid``, ``channel_id``, ``neighbor_channel_id``,
       ``seen_neighbors``, ``hello_interval``, ``hold_time``,
       ``timestamp``, and optional ``ready`` (``bool``). Response returns
       ``status`` and ``ready``.
   * - ``linkProbe`` / ``linkProbeResponse``
     - Quantum path verification. Request includes ``src_cid``,
       ``dst_cid``, ``channel_id``, and ``probe_id``; response includes
       ``status`` (``ok``, ``not_ready``, ``error``) and optional
       ``data`` (``detected``, ``counts``, ``tof``, ``vlink``, ``rx_count``,
       ``reason``).
   * - ``linkSwitchPortCheck`` / ``linkSwitchPortCheckResponse``
     - Switch port routing verification (OpticalSwitch nodes only).
       Response status is ``ok``, ``port_down``, or ``not_routed``.
   * - ``linkControl`` / ``linkControlResponse``
     - Remote REPL control commands (``show``, ``show_detail``,
       ``connect``, ``disconnect``, ``probe``, ``set``, ``switch_check``).
       See :doc:`Remote REPL </remote-repl>`.
   * - ``linkStateUpdate``
     - Monitor event schema for link state transitions and telemetry.

Monitor Events
--------------

When a link changes state—or emits periodic classical/quantum probe
telemetry—the agent publishes a ``MonitorEvent`` to the ``monitor``
topic with ``eventType: "linkStateUpdate"`` and a ``value`` object
containing:

- ``src_cid``, ``dst_cid``, ``channel_id``, ``neighbor_channel_id``,
  ``direction``, ``is_quantum`` — channel identity and directionality
- ``state`` and ``state_changed`` (``bool``) — current link state
  (``DOWN``, ``INIT``, ``CONTROL_UP``, ``QUANTUM_UP``) and whether this
  event represents a transition
- ``last_probe_sent``, ``last_probe_received``, ``last_probe_status``,
  ``last_probe_id``, ``probe_tx_count``, ``probe_rx_count``,
  ``probe_ok_count``, ``probe_fail_count`` — probe counters and latest
  timestamps
- ``last_probe_info`` and ``probe_history`` (up to 15 recent samples) —
  including ``rtt_ms`` for classical ``hello`` probes or ``tof``,
  ``counts``, ``wavelength``, ``power``, and ``vlink`` for quantum probes

The controller's :doc:`Monitoring </monitoring>` plugin persists these
events in the ``LinkState`` collection and updates the topology and
router when ``state_changed`` is ``True``. When ``QUANTNET_LINK_ADJACENCY``
is enabled on the controller, only active links (``QUANTUM_UP`` for
quantum channels; ``CONTROL_UP`` or ``QUANTUM_UP`` for classical channels)
are included for routing and scheduling, while the :doc:`Northbound API </api>`
surfaces all link states and probe streams in the dashboard.

Interactive REPL
----------------

When running locally (without ``--no-repl``), the agent drops into an
interactive REPL after startup. See :doc:`Remote REPL </remote-repl>`
for the full command reference, which applies to both the local and
remote REPL.

The local REPL additionally supports:

- ``show probe [<neighbor>:<ch>]`` — inspect optical drivers, virtual
  links, Time-of-Flight, and probe history
- ``probe <neighbor>:<ch> [count]`` — run one or more on-demand probes
  with live pass/skip/fail output
- ``debug link`` / ``no debug link`` and ``debug probe`` /
  ``no debug probe`` — toggle verbose link and probe logging
- ``logging console`` / ``no logging console`` and ``show logging`` —
  toggle and inspect console log output
- ``monitor`` and ``monitor probe`` — curses-based live dashboards (press
  ``p`` to toggle link/probe view, ``r`` to trigger probes, ``q`` to
  return to prompt)
