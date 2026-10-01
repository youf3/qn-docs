Link Adjacency Protocol
=======================

The Link Adjacency Protocol provides IS-IS-inspired neighbor discovery
and link state management for QNCP agents. Each agent actively discovers
its neighbors, establishes bilateral classical and quantum link state,
and reports state changes to the controller.

Overview
--------

When an agent starts, the ``LinkAdjacencyManager`` service reads the
node configuration to identify neighboring nodes. It then runs a
periodic Hello loop that:

1. Sends Hello messages to each neighbor over MQTT RPC.
2. Checks for bilateral confirmation (both sides see each other).
3. Runs quantum probes on confirmed links.
4. Expires links whose hold timer has elapsed.

State changes are published to the ``monitor`` topic as
``linkStateUpdate`` events, which the controller uses to filter
topology edges.

Link States
-----------

Each link transitions through four states:

.. list-table::
   :header-rows: 1
   :widths: 20 80

   * - State
     - Description
   * - ``DOWN``
     - No communication with neighbor. Initial state, or hold timer
       expired.
   * - ``INIT``
     - First Hello sent to neighbor, awaiting bilateral confirmation.
   * - ``CONTROL_UP``
     - Both sides have exchanged Hellos and see each other in their
       ``seen_neighbors`` list. Classical control path is confirmed.
   * - ``QUANTUM_UP``
     - Quantum probe succeeded. The optical/quantum path is verified
       end-to-end.

State Machine
~~~~~~~~~~~~~

::

    DOWN ──(send hello)──► INIT ──(seen by neighbor)──► CONTROL_UP ──(probe ok)──► QUANTUM_UP
     ▲                                                       │                         │
     └───────────────────(hold timer expired)────────────────┘─────────────────────────┘

Architecture
------------

The protocol is split across three components:

**LinkStateTable** (``quantnet_agent.hal.link_state_table``)
    Shared in-memory state. Stores a ``LinkEntry`` per neighbor with
    current state, timestamps, channel IDs, and transition history.
    Attached to the HAL as ``hal.link_state_table``.

**LinkAdjacencyManager** (``quantnet_agent.service.link_adjacency``)
    Service-layer component that drives the outbound Hello loop, hold
    timer checks, and quantum probes. Created during agent startup and
    attached to the HAL as ``hal.link_mgr``.

**LinkInterpreter** (``quantnet_agent.hal.interpreter.link``)
    Built-in command interpreter that handles inbound RPC commands:

    - ``link.hello`` — process incoming Hello from a neighbor
    - ``link.probe`` — respond to quantum probe requests
    - ``link.control`` — handle remote REPL commands (see
      :doc:`Remote REPL </remote-repl>`)
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
       ``src_cid``, ``dst_cid``, ``seen_neighbors``, ``hello_interval``,
       ``hold_time``, and ``timestamp``.
   * - ``linkProbe`` / ``linkProbeResponse``
     - Quantum path verification. Payload includes ``src_cid``,
       ``dst_cid``, ``channel_id``, and ``probe_id``.
   * - ``linkSwitchPortCheck`` / ``linkSwitchPortCheckResponse``
     - Switch port routing verification (OpticalSwitch nodes only).
       Response status is ``ok``, ``port_down``, or ``not_routed``.
   * - ``linkControl`` / ``linkControlResponse``
     - Remote REPL control commands. See :doc:`Remote REPL </remote-repl>`.
   * - ``linkStateUpdate``
     - Monitor event published on state transitions. Includes
       ``src_cid``, ``dst_cid``, ``state``, and ``timestamp``.

Monitor Events
--------------

When a link changes state, the agent publishes a ``MonitorEvent`` to the
``monitor`` topic with ``eventType: "linkStateUpdate"`` and a value
containing:

- ``src_cid``: the local agent's CID
- ``dst_cid``: the neighbor's CID
- ``state``: the new link state (``DOWN``, ``INIT``, ``CONTROL_UP``,
  ``QUANTUM_UP``)

The controller's monitoring plugin persists these events and can use
them to filter topology edges — only links in ``QUANTUM_UP`` state are
considered active for routing and scheduling.

Interactive REPL
----------------

When running locally (without ``--no-repl``), the agent drops into an
interactive REPL after startup. See :doc:`Remote REPL </remote-repl>`
for the full command reference, which applies to both the local and
remote REPL.

The local REPL additionally supports:

- ``debug link`` / ``no debug link`` — toggle verbose logging for the
  link adjacency subsystem
- ``monitor`` — curses-based live dashboard showing link states and
  recent transitions in real time
