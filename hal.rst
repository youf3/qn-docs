Hardware Abstraction Layer
##########################

Hardware Abstraction Layer (HAL)
--------------------------------

The Hardware Abstraction Layer (HAL) provides :doc:`Agent </agent>`
:doc:`Command Interpreters </interpreters>` with access to hardware
drivers for managing real-time hardware operations.

When an RPC command request is received, agent command interpreters
often need to interact with real-time hardware devices to perform tasks
such as local device and link calibration or entanglement generation.

HAL reads the real-time hardware configuration from a configuration file
and dynamically initializes a driver within its object instance. This
instance is then passed to the command interpreter during its
initialization.

Device Configuration
--------------------

Device configurations are specified under the ``[devices]`` section in
the configuration file and require the following parameters:

* ``enabled`` : ``bool`` - Enables or disables the device
* ``type`` : Specifies the device type (matching HAL base classes ``LightSrc``,
  ``Filter``, ``LightMeasurement``, ``SignalMeasurement``,
  ``AnalogController``, ``DigitalController``, ``ExpFramework``; typically
  configured as ``Light_Source``, ``Filter``, ``Light_Measurement``,
  ``Signal_Measurement``, ``Exp_Framework``, ``EGP``, or ``Messaging``)
* ``driver`` : Name of the driver class to be initialized
* Additional items are passed as dictionary parameters to the driver
  constructor when initialized

When emulating link probing without physical optical hardware, configure
``DummyLightSrc`` (under ``[[lightsource]]``) and ``DummySignalMeasurement``
(under ``[[signalmeasurement]]``); see
:doc:`Link Emulation & Probing </link-emulation>` for all virtual-link
parameters.

Example Device Configuration::

    [devices]
    [[exp_framework]]
    enabled = true    
    type = Exp_Framework
    driver = ArtiqClient
    host = ::1
    port = 3251

    [[lightsource]]
    enabled = true
    type = Light_Source
    driver = DummyLightSrc
    ip = 127.0.0.1
    base_port = 15000

    [[signalmeasurement]]
    enabled = true
    type = Signal_Measurement
    driver = DummySignalMeasurement
    ip = 127.0.0.1
    base_port = 15000
    window = 2.0

    [[epc]]
    enabled = true
    type = Filter
    driver = DummyEPC    

    [[polarimeter]]
    enabled = true
    type = Light_Measurement    
    driver = Thorlabs
    device = /dev/usbtmc0

    [[egp]]
    enabled = true
    type = EGP
    driver = SimEGPDriver
    protocol = EGProtocol

    [[messaging]]
    enabled = true
    type = Messaging
    driver = PassthroughDriver

Link State Attributes
---------------------

The HAL exposes two additional attributes for the
:doc:`Link Adjacency Protocol </link-adjacency>`:

``hal.link_state_table``
    A ``LinkStateTable`` instance storing per-channel link state keyed by
    ``<neighbor_cid>:<channel_id>``. Each entry is a ``LinkEntry``
    dataclass with fields: ``state``, ``last_hello_sent``,
    ``last_hello_received``, ``hold_time``, ``neighbor_cid``,
    ``channel_id``, ``neighbor_channel_id``, ``switch_in_path``,
    ``switch_cid``, ``is_quantum``, ``direction``, ``neighbor_ready``,
    ``history``, ``last_probe_sent``, ``last_probe_received``,
    ``last_probe_status``, ``last_probe_id``, ``probe_tx_count``,
    ``probe_rx_count``, ``probe_ok_count``, ``probe_fail_count``,
    ``last_probe_info``, and ``probe_history``.

``hal.link_mgr``
    The ``LinkAdjacencyManager`` instance that drives the Hello loop,
    hold timer checks, and classical/quantum probes. Provides methods:
    ``connect(key)``, ``disconnect(key)``,
    ``set_hello_interval(n)``, ``set_hold_time(n)``.
