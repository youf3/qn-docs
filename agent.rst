===========================
QUANT-NET Agent
===========================

.. image:: https://img.shields.io/pypi/v/quantnet_agent.svg
        :target: https://pypi.python.org/pypi/quantnet_agent


.. github-shield::
    :last-commit:
    :repository: qn-agent
    :branch: main

The QUANT-NET Agent that registers resources, interfaces with devices, and interprets protocol commands from the Controller.


.. click:: quantnet_agent.cli:main
   :prog: quantnet_agent
   :nested: full

Monitoring Events
-----------------

The agent scheduler publishes ``MonitorEvent`` messages to the ``monitor``
topic as tasks complete. Each event includes:

- ``eventType``: ``"agentTaskResult"`` — identifies the message as a local
  task result (distinct from a full experiment result).
- ``value.name``: the task name.
- ``value.exp_id``: the experiment/task ID, used by the controller to
  correlate results when queried via ``getTasks``.
- ``value.result``: the task result payload (when present).

Configuration
-------------

The agent reads its configuration from an INI file. A path can be provided
explicitly with the ``-c``/``--config`` CLI flag; if the file does not exist
the agent exits immediately with an error.

Without ``-c``, the config path depends on the environment:

 * If ``$QUANTNET_HOME`` is set: ``$QUANTNET_HOME/etc/agent.cfg``
 * Otherwise: ``/etc/quantnet/agent.cfg``

If no config file is found the agent continues with defaults and logs a
warning. Message broker log output is suppressed to warning level by default.

