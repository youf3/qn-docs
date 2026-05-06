=========
Changelog
=========

1.1.0 (2026-04)
------------------

qn-api (new)
~~~~~~~~~~~~
* Added northbound API server and web dashboard for experiment management and topology visualization.
* API server provides REST endpoints for querying topology, managing experiments, and retrieving results.
* :doc:`Northbound API </api>` documentation added to the docs.

qn-mq
~~~~~

* Add ``EventType`` string enum with typed constants for all monitor event types
  (``agentState``, ``experimentResult``, ``agentHeartbeat``,
  ``agentTaskSchedulerPhase``, ``agentTaskSchedulerTask``, ``agentTaskResult``).
* Add ``agentTaskResult`` as a monitor event type in the message schema.
* Add RPC message types for querying agent task results from the controller
  (``MonitorTask`` / ``MonitorTaskResponse``).
* Add a ``length`` property to the ``Channel`` schema, exposing quantum channel
  length in topology data.
* Compatible with Python 3.10+.

qn-server
~~~~~~~~~

* Add agent heartbeat validation: agents not heard from within 60 seconds are
  excluded when matching nodes to an experiment path.
* Enhanced topology API: full mode returns complete node definitions including
  channel data, plus aggregate node, qubit, and channel counts.
* Monitor plugin tracks heartbeat events to maintain each node's last-seen
  timestamp in the database.
* Monitor plugin adds a ``getTasks`` RPC command for querying recorded agent
  task results.
* Experiment protocol separates local task results from experiment-level
  results; supports filtering queries by agent.
* Routing fixes: correctly handle mixed-direction graph topologies; recover
  quantum links when graph conversion drops edges in multi-graph topologies.
* Configuration refactoring: centralized configuration management with explicit
  config file selection via the ``-c``/``--config`` CLI flag; exits with an
  error if the specified file does not exist.
* Config file search path updated — ``$QUANTNET_HOME/etc/quantnet.cfg``
  (when ``$QUANTNET_HOME`` is set) then ``/opt/quantnet/etc/quantnet.cfg``
  (always); a previously probed path has been removed.
* Database and message broker initialization decoupled from global
  configuration.
* Calibration and experiment requests now follow separate execution paths,
  each using its own protocol and schema when communicating with agents.
* Request parameters are now structured and typed before being forwarded
  to agents.
* Allocation and result-retrieval errors now identify the specific agent
  that failed.

qn-agent
~~~~~~~~

* Scheduler monitor events now use the ``agentTaskResult`` event type
  (previously ``experimentResult``) and include the experiment ID in the
  event value.
* Configuration refactoring: centralized config management with auto-discovery;
  ``-c``/``--config`` flag added, mirroring the controller pattern. Explicit
  ``-c`` path exits with error if not found.
* Config file search path: ``$QUANTNET_HOME/etc/agent.cfg`` if
  ``$QUANTNET_HOME`` is set, otherwise ``/etc/quantnet/agent.cfg``.
* Message broker log output suppressed to warning level by default.

1.0.0 (2025-12-05)
------------------

* Initial packaged release.
