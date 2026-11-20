============================
Monitoring Module
============================

Monitoring Plugin
-----------------

The :doc:`Controller </server>` provides quantum network monitoring
capabilities through a monitoring plugin interface, which defines
handlers for monitoring messages. A monitoring plugin must implement the
abstract method ``handle_resource_update()`` to manage monitoring events
received from a ``msgserver`` that listens to a chosen topic.

The built-in monitoring plugin handles monitoring events received by the
controller. It subscribes to the ``monitor`` topic for ``MonitorEvent``
messages. Upon receiving a message, the plugin:

1. Validates the message format against the monitoring message schema.
2. Dispatches on ``eventType``:

   - **agentHeartbeat** — updates the node's last-seen timestamp in the
     database (not persisted to the Monitor DB).
   - **agentState** — appended to the capped ``MonitorState`` collection;
     if the state is ``UNKNOWN`` or ``DEAD``, clears ``last_seen`` and
     marks incident links ``DOWN`` in ``LinkState``.
   - **linkStateUpdate** — upserted into the ``LinkState`` collection
     keyed by ``(src_cid, dst_cid, channel_id)`` (see
     `Link Connection Status Monitoring`_ below).
   - **agentTaskResult**, **experimentResult**, and all other event types —
     persisted to the Monitor database.

Protocol
--------

The :doc:`Message Bus </comms>` defines a built-in monitoring message
format in JSON, which includes the following fields:

- ``rid``: ``str`` — Resource ID (e.g. agent identifier)
- ``ts``: ``float`` — UTC Unix timestamp
- ``eventType``: ``str`` — Type of event (see ``EventType`` below)
- ``value``: ``str | int | float | object`` — Value of the monitored resource

Event Types
-----------

All valid event types are defined in the ``EventType`` enum exported by qn-mq
(see :doc:`Message Bus </comms>`):

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - Event Type
     - Description
   * - ``agentState``
     - Agent state transition (e.g. Not Ready / Ready). Persisted to the
       Monitor DB.
   * - ``experimentResult``
     - Result of a completed experiment run. Persisted to the Monitor DB.
   * - ``agentHeartbeat``
     - Periodic liveness ping from an agent. Updates the node's last-seen
       timestamp; not stored in the Monitor DB.
   * - ``agentTaskSchedulerPhase``
     - Phase update from the agent task scheduler. Persisted to the Monitor DB.
   * - ``agentTaskSchedulerTask``
     - Individual task status from the agent task scheduler. Persisted to the
       Monitor DB.
   * - ``agentTaskResult``
     - Result of a completed agent local task, including the experiment ID and
       result payload. Persisted to the Monitor DB and queryable via ``getTasks``.
   * - ``linkStateUpdate``
     - Link adjacency state transition (``DOWN``, ``INIT``, ``CONTROL_UP``,
       ``QUANTUM_UP``) or periodic probe telemetry update. Published by the
       agent's :doc:`Link Adjacency Protocol </link-adjacency>` and upserted
       into the ``LinkState`` collection.

Link Connection Status Monitoring
---------------------------------

The Monitor plugin tracks per-channel link health and probe telemetry
across the network:

1. **``LinkState`` Persistence**: Incoming ``linkStateUpdate`` events are
   upserted into the ``LinkState`` MongoDB collection keyed by
   ``(src_cid, dst_cid, channel_id)``. Both state transitions
   (``state_changed=True``) and periodic probe snapshots
   (``state_changed=False``) update the link document with ``state``,
   ``timestamp``, ``is_quantum``, ``direction``, probe counters
   (``probe_tx_count``, ``probe_rx_count``, ``probe_ok_count``,
   ``probe_fail_count``), ``last_probe_info``, and ``probe_history``.
2. **Topology & Router Refresh**: When ``state_changed`` is ``True``, the
   plugin marks the :doc:`Controller </server>` resource manager topology
   cache dirty (``rm.set_topo_updated()``). For routing-significant states
   (``CONTROL_UP``, ``QUANTUM_UP``, ``DOWN``), it also triggers
   ``router.refresh()`` so path computation reflects live link status.
3. **Agent Liveness & Automatic Link Down**:

   - On clean agent deregistration (``agentDeregister``) or an
     ``agentState`` event of ``UNKNOWN`` or ``DEAD``, the controller clears
     ``last_seen = 0`` and forces all links where the agent is ``src_cid``
     or ``dst_cid`` to ``DOWN``.
   - Separately from the 60-second scheduler heartbeat check below,
     ``ResourceManager._is_node_stale()`` and the
     :doc:`Northbound API </api>` enforce a **20-second**
     ``AGENT_LIVENESS_TIMEOUT``: if either endpoint's ``last_seen`` is
     cleared or older than 20 seconds, link queries evaluate incident
     links as ``DOWN``.

Agent Heartbeat Tracking
------------------------

The monitor plugin uses incoming ``agentHeartbeat`` events to maintain a
last-seen timestamp for each node in the database. This timestamp is used
by the controller to exclude stale agents when matching nodes to an
experiment path (see :doc:`Scheduling </scheduling>`).

A node is considered stale for scheduling if no heartbeat has been
received within 60 seconds (and stale for live topology/link-state checks
after 20 seconds).

getTasks RPC
------------

The monitor plugin exposes a ``getTasks`` RPC command that allows clients to
query recorded agent task results from the Monitor database. The request
accepts an optional agent filter to narrow results to a specific node.

Each entry in the response includes the task and experiment identifiers,
the result payload, status, timestamps, and the name of the originating task.
