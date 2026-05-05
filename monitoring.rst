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
   - **agentState**, **agentTaskResult**, **experimentResult**, and all other
     event types — persisted to the Monitor database.

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

Agent Heartbeat Tracking
------------------------

The monitor plugin uses incoming ``agentHeartbeat`` events to maintain a
last-seen timestamp for each node in the database. This timestamp is used
by the controller to exclude stale agents when matching nodes to an
experiment path (see :doc:`Scheduling </scheduling>`).

A node is considered stale if no heartbeat has been received within 60 seconds.

getTasks RPC
------------

The monitor plugin exposes a ``getTasks`` RPC command that allows clients to
query recorded agent task results from the Monitor database. The request
accepts an optional agent filter to narrow results to a specific node.

Each entry in the response includes the task and experiment identifiers,
the result payload, status, timestamps, and the name of the originating task.
