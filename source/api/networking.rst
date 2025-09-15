Off-Device Communication
------------------------

|geisa-api-hdr|


GEISA allows applications to make use of 2 types of off-device communications: Message based, and IP socket based.


Message based via LwM2M
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This type of communication is available on every device with :doc:`/adm` and uses the administrative LwM2M connection for transport.

Delivery of messages may be significantly delayed or lost. The :doc:`/adm` is responsible for queuing these messages and reporting back to the Application if and when they are delivered.

Applications use the message bus to send and receive these messages to/from the platform which in turn makes use of LwM2M to transport them.  The applications themselves are unaware of any LwM2M details.

This type of communication is bi-directional and messages can be sent unsolicited by either side.

.. note::

  Messages that the application sends are deposited into an operator run system for retrieval by the operator or application developer using APIs to the operator's cloud services.  The operator or application developer can also initiate messages to the application in bulk or individually.


IP socket based to local devices, private clouds, or public clouds
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Applications can also make use of traditional IP socket based real-time communications on devices equipped with cellular, WiFi, or other IP connectivity. Applications MUST specify the endpoints they need to communicate to (IP/Port) in their Application Manifest so that those policies can be approved by operators and implemented in the platform using firewall rules.

Applications MUST also specify the expected volume of data per day per destination class.

Once a network interface is online, the Application may use outbound-initiated AF_INET/AF_INET6 sockets from within the container environment to reach the specified endpoints.

Inbound connections towards the Application are not supported in this version of GEISA as inbound port mappings can conflict between applications and accepting inbound connections open the Application to additional security concerns.

.. note::

  Support for IP socket communication is NOT REQUIRED by a GEISA device, however certain applications may REQUIRE it to function and would not be compatible with those devices.  Applications can be designed to require IP socket communication or require it but fallback in a degraded functionality mode to message based communication.


Interface Types
^^^^^^^^^^^^^^^

The method of connectivity that a device provides can vary from a home WiFi providing Internet connectivity, to private WiFi for partner integrations, or to a variety of operator maintained connectivity options such as shared WiFi hotspots, cellular, mesh, wired, or proprietary interfaces.

Applications need to know which (or multiple) of these are available and online at any given time however enumerating both the technology and the allowable use cases that they can transport can become complex quickly.

Example types of network interfaces:

- WiFi - Home (device is client)
- WiFi - Meter (device is AP)
- WiFi - Operator/Partner-Hotspot (device is client)
- Cellular - Internet APN
- Cellular - Private APN
- Ethernet - Home
- Ethernet - Private/Partner Devices 
- Field Area Network

Other network interfaces that may be available (but out of scope of *IP socket* based communication in GEISA at this time) are:

- Bluetooth / Bluetooth LE
- IoT integrations via 802.15.4 technologies (Zigbee, Thread, etc...)
- Other wired connectivity or peripherals: RS232/485, USB connected devices, etc...


Destination Classes
^^^^^^^^^^^^^^^^^^^

The device may have zero, one, or multiple interfaces online at any given time.  Applications are not required to know these details but instead only need to know if their desired endpoints are reachable and what volume limits are in place.

Each endpoint an application defines falls into one of these categories:

- Internet Endpoint (ex: Public Cloud)
- Operator Endpoint (ex: Private Cloud)
- Local Endpoint (something off-device, but on-net and private)
- Message based (LwM2M)

For each of these classes, applications have a defined volume limit for when that class is metered.


Network State
^^^^^^^^^^^^^

For each destination class, applications have a network state that tells the application if and how much communication is available as described in :doc:`status`.

An application can request communication with more than one class of endpoint and would need a network status indicator for each class separately.  For example, it may communicate with both Internet Endpoints as well as Local Endpoints.

Applications may report statistics and logs to the operating environment for troubleshooting and logging.  For example, if the operating environment has reported that Internet Endpoints should be reachable but the Application cannot reach any, it may report this error.


Volume Limits
^^^^^^^^^^^^^

Each destination class can be metered depending on which underlying technology transports the data.  A Home WiFi would normally be considered unlimited where as a cellular connection would be metered to keep the device under a monthly volume limit.

An application developer MUST define volume limits per destination class in their Application Manifest.  These limits may be overridden by the operator at deployment time when converting the Application Manifest into a Deployment Manifest.

These volume limits are specified as a per day (24 hour period) limit in bytes.  Both transmit and receive data counts toward the application's limit.  The operator may define a daily rollover mechanism, and a reset period (ex: day of the month).  GEISA does not define if or how header and encapsulation bytes count towards volume limits.

The application obtains the remaining volume limits and when the next reset occurs via :doc:`/api/status`.

If an application exhausts its volume quota for one or more destination classes, it will be sent a network state update with the volume field set to *zero* for those classes.  When this condition is cleared (on the next day or reset period), the application will be sent another network status update returning the volume field to *metered*

Volume limits do not apply to destination classes when the volume field set to *unlimited*.


Security considerations
^^^^^^^^^^^^^^^^^^^^^^^

For message based communications, the operating environment will provide encryption and authentication for data passed between the device and head-end via LwM2M, applications MAY perform further encryption and/or authentication on top of what :doc:`/adm` provides.

For IP socket based communications, the application is responsible for encryption and authentication of data passed between the device and its endpoints where needed.


Connectivity
^^^^^^^^^^^^

The operating environment is responsible for providing network connectivity between each Application container environment and network interfaces.

The platform is responsible for both implementing policy (by firewall and forwarding rules), and providing connectivity between these components.

.. note::

  GEISA does not mandate a specific technology that the implementer of the operating environment must use to accomplish this, but does recommend the use of Linux network namespaces, veth interfaces, and iptables/nftables for filtering, NAT, and accounting.  The implementer may also make use of on-device transparent proxies if desired, however the Application must be able to use AF_INET/AF_INET6 sockets from within the container environment with any encoding and protocol within.

The operating environment MUST provide a lo interface within the container environment for each application. The lo interface must be up and configured with both 127.0.0.1 and ::1 addresses.


Policy Rules
^^^^^^^^^^^^

The :doc:`/adm/manifests` include a set of Endpoints the Application is expecting to communicate with.  The application must list every endpoint that it wishes to communicate with per destination class.

DNS
^^^

.. warning:: 

  TODO: should we forgo DNS for GEISA 1.0?

While policy rules are defined using IP addresses, Applications may use DNS queries to avoid hard-coded IP literals within their application source or configuration files.

The Operating Environment and Network manager must provide DNS services for applications to use, however the scope of resolution may be limited particularly for devices that are not or are poorly Internet connected.

GEISA highly recommends that the Operating Environment implement a caching local resolver that honors TTL to reduce network traffic off-device due to repeated application lookups for the same name.

The application must be able to reach DNS services from within the container environment by using standard Linux libraries (i.e.: libnss/resolvconf/etc...)

DNS in a multi-tenant and multi-interface environment can get quite complex.  For example, an operator may implement their Operator Endpoints using a dedicated private TLD and configure the resolver to direct DNS lookups for that TLD over their private network where other TLDs for Internet Endpoints use a home WiFi or Internet connected Cellular.




Local Endpoint Considerations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. warning:: 

  TODO: should we forgo Local endpoint connectivity for GEISA 1.0?

Local Endpoints are defined to allow an Application to access local resources on a connected network.  Typically this would be for local device integration such as smart plugs, EVSE, Solar Inverter and battery storage equipment and so on.  This type of class is typically only available for devices that are connected to a home WiFi, a Meter hosted WiFi, or wired connection.

As the IP addressing of the connected network are not known and not static between devices, the Application Manifest cannot list destination/source IP addresses for policy rules.

Many local communication devices use mDNS for discovery, directed broadcasts, multicast, or link-local addressing.  How this is implemented is TBD







MQTT Details
=============

- QoS: 1 / Acknowledged R/R
- Req Topic: ``geisa/api/message-req/<userid>``
- Rsp Topic: ``geisa/api/message-rsp/<userid>``


API Permissions
================

- Application:

  - Publish: ``geisa/api/message-req/<userid>``
  - Subscribe: ``geisa/api/message-rsp/<userid>``

- Platform:

  - Wildcard Subscribe: ``geisa/api/message-req/*``
  - Publish: ``geisa/api/message-rsp/<userid>``


Transaction Data
=================

.. warning:: 
  
  Need to add refererence to content within |geisa-schemas-repo| here.




|geisa-pyramid|
