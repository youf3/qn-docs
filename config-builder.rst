Config Builder
==============

``qn-config-builder`` generates all configuration artifacts needed to
deploy a QNCP network from a high-level topology description. It
supports both single-host (local) and multi-host (distributed)
deployment modes.

Installation
------------

.. code-block:: bash

    pip install -e qn-config-builder

CLI Usage
---------

Local Deployment
~~~~~~~~~~~~~~~~

Generate a single ``docker-compose.yml`` with all services:

.. code-block:: bash

    # 2-QPU full-mesh (default)
    qn-config-builder generate full-mesh -o ./demo/

    # 3-QPU linear chain
    qn-config-builder generate linear-direct -n 3 -o ./demo/

    # Run it
    cd demo && docker compose up -d

Distributed Deployment
~~~~~~~~~~~~~~~~~~~~~~

Generate per-VM bundles for multi-host deployment (e.g., FABRIC
testbed):

.. code-block:: bash

    qn-config-builder generate full-mesh -n 2 \
        --distributed --controller-ip 10.0.0.1 \
        -o ./dist-demo/

This produces:

.. code-block:: text

    dist-demo/
    ├── controller/
    │   ├── conf/
    │   │   ├── quantnet.cfg        # MQ host = mosquitto (local)
    │   │   ├── agent.cfg
    │   │   └── mosquitto.conf
    │   └── docker-compose.yml      # mosquitto + mongo + controller + api
    ├── agents/
    │   ├── QPU-1/
    │   │   ├── conf/
    │   │   │   └── agent.cfg       # MQ host = 10.0.0.1 (controller)
    │   │   ├── node_defs/
    │   │   │   └── conf_QPU-1.json
    │   │   └── docker-compose.yml  # single agent, network_mode: host
    │   ├── QPU-2/
    │   │   └── ...
    │   └── BSM-1_2/
    │       └── ...
    └── topology.yaml

Each bundle is self-contained — upload it to the target VM and run
``docker compose up -d``.

Key differences from local mode:

- Controller compose runs mosquitto, mongo, controller, and API on a
  Docker bridge network. MQTT port 1883 is exposed on ``0.0.0.0`` so
  remote agents can connect.
- Agent compose uses ``network_mode: host`` so the agent container can
  reach the remote MQTT broker directly.
- Agent ``agent.cfg`` has ``host=<controller-ip>`` instead of
  ``host=broker``.
- Agents run with ``--no-repl`` for headless Docker operation. Use
  :doc:`quantnet_repl </remote-repl>` to connect interactively.

YAML Descriptor
~~~~~~~~~~~~~~~

For full control, write a YAML descriptor and build from it:

.. code-block:: bash

    qn-config-builder init full-mesh -n 3 -o topology.yaml
    # edit topology.yaml ...
    qn-config-builder build topology.yaml -o ./demo/

Minimal descriptor:

.. code-block:: yaml

    topology:
      template: full-mesh
    nodes:
      qpu:
        count: 3

Distributed descriptor:

.. code-block:: yaml

    topology:
      template: full-mesh
    nodes:
      qpu:
        count: 2
    deployment:
      mode: distributed
      controller_address: "10.0.0.1"

Interactive Wizard
~~~~~~~~~~~~~~~~~~

.. code-block:: bash

    qn-config-builder wizard

The wizard prompts for topology template, QPU count, qubit
configuration, deployment mode, and output directory.

Python API
----------

.. code-block:: python

    from qn_config_builder.models import Descriptor
    from qn_config_builder.builder import build_deployment
    from pathlib import Path

    # Local deployment
    descriptor = Descriptor(
        topology={"template": "full-mesh"},
        nodes={"qpu": {"count": 3}},
    )
    build_deployment(descriptor, Path("./demo"))

    # Distributed deployment
    descriptor = Descriptor(
        topology={"template": "full-mesh"},
        nodes={"qpu": {"count": 2}},
        deployment={
            "mode": "distributed",
            "controller_address": "10.0.0.1",
        },
    )
    build_deployment(descriptor, Path("./dist-demo"))

Topology Templates
------------------

.. list-table::
   :header-rows: 1
   :widths: 20 30 50

   * - Template
     - Nodes Created
     - Pattern
   * - ``full-mesh``
     - N QPUs + N*(N-1)/2 BSMs
     - Every QPU pair connected via a dedicated BSM
   * - ``linear-direct``
     - N QPUs + (N-1) BSMs
     - Adjacent QPU pairs connected via BSM chain
   * - ``linear-switched``
     - N QPUs + N switches + (N-1) BSMs [+ N MNodes]
     - QPU → Switch → BSM ← Switch ← QPU

Descriptor Reference
--------------------

.. list-table::
   :header-rows: 1
   :widths: 25 15 60

   * - Field
     - Default
     - Description
   * - ``topology.template``
     - ``full-mesh``
     - Topology template name
   * - ``nodes.qpu.count``
     - ``2``
     - Number of QPU nodes
   * - ``nodes.qpu.prefix``
     - ``QPU``
     - QPU node ID prefix
   * - ``nodes.qpu.defaults.qubits.communication``
     - ``2``
     - Communication qubits per QPU
   * - ``nodes.qpu.defaults.qubits.data``
     - ``0``
     - Data qubits per QPU
   * - ``nodes.mnode.enabled``
     - ``false``
     - Enable measurement nodes (linear-switched only)
   * - ``links.loss``
     - ``{value: 5, unit: dB}``
     - Channel loss
   * - ``links.length``
     - ``{value: 1, unit: km}``
     - Channel length
   * - ``server.mq_host``
     - ``broker``
     - MQTT broker hostname
   * - ``server.database``
     - ``mongodb://mongo:27017``
     - MongoDB connection string
   * - ``docker.image_tag``
     - ``develop``
     - Docker image tag for QNCP images
   * - ``docker.registry``
     - ``ghcr.io/quant-net``
     - Docker image registry
   * - ``docker.api_port``
     - ``8081``
     - API dashboard port
   * - ``deployment.mode``
     - ``local``
     - ``local`` or ``distributed``
   * - ``deployment.controller_address``
     - (empty)
     - Controller IP/hostname (required for distributed)

Import Existing Configs
-----------------------

Import an existing set of ``conf_*.json`` node definitions into a YAML
descriptor:

.. code-block:: bash

    qn-config-builder import ./existing-node-defs/ -o topology.yaml

    # Optionally rebuild from the imported descriptor
    qn-config-builder import ./existing-node-defs/ -o topology.yaml --build ./demo/
