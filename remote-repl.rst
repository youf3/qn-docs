Remote REPL
===========

The ``quantnet_repl`` CLI tool provides interactive remote control of
any running QNCP agent over MQTT. Multiple users can connect to the
same or different agents simultaneously — each session is independent.

.. click:: quantnet_agent.service.remote_repl:main
   :prog: quantnet_repl
   :nested: full

Quick Start
-----------

Connect to an agent running on a FABRIC deployment:

.. code-block:: bash

    # SSH into any VM with MQTT access, then:
    quantnet_repl -a QPU-1 --mq-host <controller-ip>

Or locally during development:

.. code-block:: bash

    quantnet_repl -a QPU-1 --mq-host localhost

The tool connects to the MQTT broker, probes the target agent for its
current link table, and presents an interactive prompt:

.. code-block:: text

    Connected to MQTT broker at localhost:1883
    Target agent: QPU-1
    Neighbors: BSM-1_2

    Type 'help' for commands.

    quantnet-agent [QPU-1]>

If the agent is not yet running, the REPL prints a warning and retries
on the first command.

Commands
--------

.. list-table::
   :header-rows: 1
   :widths: 35 65

   * - Command
     - Description
   * - ``show link``
     - Show all links and their current state.
   * - ``show link <neighbor>``
     - Detailed view of a single link, including channel IDs, hold
       time, last hello timestamps, and transition history.
   * - ``connect <neighbor>``
     - Manually trigger link adjacency with a neighbor (sends Hello,
       transitions from DOWN to INIT).
   * - ``disconnect <neighbor>``
     - Bring a link to DOWN state and publish a state update.
   * - ``probe <neighbor>``
     - Trigger a quantum probe on a link.
   * - ``set hello-interval <n>``
     - Change the Hello interval (seconds) at runtime.
   * - ``set hold-time <n>``
     - Change the hold time (seconds) at runtime.
   * - ``show switch-check <neighbor>``
     - Query the optical switch port status for a link (switched
       topologies only).
   * - ``monitor``
     - Enter a curses-based live dashboard that polls link state every
       second. Press ``q`` to return to the prompt.
   * - ``help``
     - Print the command reference.
   * - ``exit`` / ``quit``
     - Disconnect from the MQTT broker and exit.

Example Session
~~~~~~~~~~~~~~~

.. code-block:: text

    quantnet-agent [QPU-1]> show link

    Neighbor                                      State          Last RX
    ---------------------------------------------------------------------------
    BSM-1_2                                       QUANTUM_UP     3s ago

    quantnet-agent [QPU-1]> show link BSM-1_2

    Link: BSM-1_2
      State:               QUANTUM_UP
      Channel:             1 → 4
      Switch in path:      False
      Hold time:           30s
      Last hello RX:       3s ago
      Last hello TX:       2s ago
      Recent transitions:
        14:23:01  INIT
        14:23:11  CONTROL_UP
        14:23:21  QUANTUM_UP

    quantnet-agent [QPU-1]> disconnect BSM-1_2
    Disconnecting BSM-1_2... OK

    quantnet-agent [QPU-1]> connect BSM-1_2
    Triggering adjacency with BSM-1_2... OK

How It Works
------------

The remote REPL communicates with the agent via a single multiplexed
RPC command: ``link.control``. Each user command is translated into a
``linkControl`` message with an ``action`` field (``show``, ``connect``,
``disconnect``, ``probe``, ``set``, ``switch_check``) and sent to the
agent's RPC topic (``rpc/<agent-id>``).

The agent's ``LinkInterpreter`` handles the ``link.control`` RPC,
dispatches the action, and returns a ``linkControlResponse`` with the
result.

Since each ``quantnet_repl`` instance creates its own MQTT client with
a unique correlation ID, multiple users can connect simultaneously
without interference.

Headless Agent Mode
-------------------

By default, agents in Docker containers run with ``--no-repl`` so they
don't attempt to read from stdin. The ``quantnet_repl`` tool is the
intended way to interact with deployed agents.

For local development, omit ``--no-repl`` to get the embedded
interactive REPL directly in the agent's terminal:

.. code-block:: bash

    # Local dev — embedded REPL
    quantnet_agent -a QPU-1 -n conf_QPU-1.json

    # Docker / production — headless
    quantnet_agent -a QPU-1 -n conf_QPU-1.json --no-repl

Tab Completion
--------------

Both the local and remote REPL support readline tab completion for
commands and neighbor CIDs. Press ``Tab`` to autocomplete.
