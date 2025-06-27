# 24.June - Mnemosyne Telemetry Logger Design

## Mnemosyne Telemetry Logger Design
Mnemosyne  is  a  modular  Python-based  telemetry  logger  designed  for  industrial  environments.  It
automatically discovers machines on a given network subnet using common Ethernet-based industrial
protocols, then collects and logs their telemetry data in a structured SQL database. The system is built to
run on a standard developer laptop (no special hardware required) and emphasizes extensibility, high-
throughput data capture, and easy data access for digital twin and predictive maintenance applications. In
the  context  of  OrionFlow’s  Digital  Twin  and  RAG-based  (Retrieval-Augmented  Generation)  predictive
maintenance workflows, Mnemosyne provides a critical data ingestion layer that integrates all machine
data  for analysis. Key aspects of the design include network scanning and device classification per protocol,
a  plugin  architecture  for  protocol-specific  modules,  automatic  telemetry/tag  discovery,  high-frequency
asynchronous logging to an SQL database (accessible via tools like DBeaver), a developer-friendly CLI, and
hooks for future protocol expansions and cloud integrations.
## Network Scanning and Device Discovery
Mnemosyne must  scan the local subnet  to find any industrial devices, using protocol-specific discovery
methods for each targeted protocol. The system will classify devices by protocol  based on their responses.
The scanning approach for each protocol is as follows:
Modbus-TCP:  Mnemosyne scans the IP range (e.g. using a ping sweep or trying connections) for
devices listening on the standard Modbus port 502/TCP. For each host with port 502 open, it will
attempt a Modbus connection and use a lightweight identification request to confirm the device.
Specifically, it can issue the Modbus Read Device Identification  command (Function 43 / 14) . If
the device supports this function, it returns metadata like vendor name, product code, and version
, which Mnemosyne can use to classify the device (e.g. PLC, sensor , energy meter , etc.). Even
if the device doesn’t support this identification function (not all Modbus devices do ), an open
port 502 with a valid Modbus response is enough to treat it as a Modbus device.
OPC UA:  Mnemosyne uses OPC UA’s built-in discovery mechanisms to find servers. OPC UA supports Local Discovery Server (LDS)  and mDNS-based discovery (Multicast DNS/Service Discovery) that
allow clients to find OPC UA servers advertising on the subnet . Mnemosyne’s OPC UA plugin can
perform an mDNS query for OPC UA service announcements or use an LDS if one is known. In
absence of these, Mnemosyne can fall back to scanning common OPC UA ports (like 4840/tcp, the
default) on hosts. When an OPC UA server is found, the plugin will retrieve its Endpoint Description  to
confirm it’s an OPC UA server . OPC UA discovery eliminates manual configuration  by automatically
advertising and finding servers , so Mnemosyne leverages that to identify all OPC UA devices.
EtherNet/IP  (CIP):  For  Rockwell/Allen-Bradley  and  other  CIP-compatible  devices,  Mnemosyne
implements the EtherNet/IP  List Identity  protocol to discover devices. EtherNet/IP devices typically
listen on UDP port 44818 for discovery queries. The Mnemosyne CIP plugin sends a broadcast
ListIdentity  request on the subnet; any CIP-enabled device will respond with its identity information
(including device name, vendor , product code, IP address) . This approach finds PLCs, drives,
and other EtherNet/IP devices without needing to probe each IP. It’s the same mechanism used by
vendor tools (e.g. Rockwell’s RSLinx or “Ferret” tool) to scan networks. Once identified, Mnemosyne
classifies the device (for example, distinguishing Allen-Bradley Logix PLCs vs. generic CIP devices).
The plugin may also scan TCP port 44818 on hosts as a fallback, to catch devices that didn’t respond
to  the  broadcast  (some  EtherNet/IP  devices  might  not  respond  to  broadcast  due  to  network
settings). Using an existing Python library like pycomm3  can simplify this – pycomm3 's CIP driver
supports device discovery and handles the underlying protocol details .
EtherCAT:  EtherCAT devices do not use IP addressing for slave devices (they communicate via a real-
time  Ethernet  bus).  However ,  if  the  laptop  is  connected  to  an  EtherCAT  network  (via  the  same Ethernet interface), Mnemosyne’s EtherCAT plugin can act as a simple EtherCAT master  to discover
slaves. Using a library such as PySOEM  (Python wrapper for the Simple Open EtherCAT Master), the
plugin  opens  the  network  interface  in  raw  mode  and  performs  a  bus  scan
(master.config_init() ) to enumerate all connected EtherCAT slaves . This yields a list of
devices with basic info like each slave’s name and vendor ID. Each found EtherCAT device can then be
queried for more info (e.g. via CoE – CANopen over EtherCAT – to get its object dictionary). EtherCAT scanning may require running Mnemosyne with appropriate privileges (raw socket access) but  no
special hardware  beyond the standard NIC is needed, as many NICs support EtherCAT frames in
software. If the NIC or OS cannot be used as an EtherCAT master , Mnemosyne will document this
limitation; but conceptually, the plugin is ready to integrate with a software master stack to find
devices on the EtherCAT bus.
PROFINET:  Mnemosyne  uses  the  PROFINET  Discovery  and  Configuration  Protocol  (DCP)  for
scanning. PROFINET DCP works at the Ethernet link layer to identify devices. The plugin sends a
broadcast DCP “Identify”  request to the standard PROFINET discovery multicast address (01:0E:CF:
00:00:00). All PROFINET-capable devices on the subnet will respond with their device information
(device name, IP, vendor , device ID). This is a known technique (also used by tools like Siemens
Proneta or third-party scripts) to discover PROFINET devices . In fact, an Nmap script exists that
employs  DCP  Identify  to  list  PROFINET  devices  on  a  network .  After  the  identify  phase,  the
PROFINET plugin can also attempt a connection on the PROFINET context (e.g. connecting to the
device’s RPC endpoint on port 34964/tcp, used for PROFINET IO communication) as a way to further
confirm the device and retrieve information. This two-step approach (DCP at layer 2, then optional
TCP  probe)  ensures  all  PROFINET  controllers  and  devices  are  found  even  if  they  haven’t  been
assigned  an  IP  or  name  yet  (since  DCP  can  find  devices  by  MAC).  Once  a  device  responds,
Mnemosyne records its name, IP, and type for classification.
Each protocol’s scanning routine is implemented in its plugin module . On startup, Mnemosyne will run
all enabled plugin scanners in parallel (or in sequence, configurable) across the target subnet or interface.
The result of scanning is a list of discovered devices, each tagged with the protocol and basic identity info.
The system avoids false positives by using protocol-specific handshakes (ensuring that, for example, a device on port 502 is truly speaking Modbus by sending a simple request). Network scanning can be
configured via the CLI (e.g. specify subnet range, or use the local interface’s subnet by default). The design
does not assume any prior knowledge of device IPs or types – it systematically searches and classifies everything on the network

## Modular Plugin Architecture for Protocol Handling
Mnemosyne follows a plugin architecture  where each industrial protocol is handled by a separate module
(or “connector”) that adheres to a common interface. This modular design allows protocols to be added or
updated independently, and enables the system to load only the needed protocol logic for the devices
present. The key elements of this architecture are:
Plugin Modules:  Each protocol resides in its own module or package under a plugins/  directory.
For  example,  there  might  be  modules  like  plugins/modbus.py ,  plugins/opcua.py , 
plugins/ethernetip.py ,  plugins/ethercat.py ,  plugins/profinet.py , etc. In complex
cases, a plugin can be a package (folder) with multiple files (e.g. plugins/opcua/__init__.py , 
client.py , etc., especially if using third-party libraries). The  CNC example  (Mach4 via Modbus-TCP) is implemented as a plugin under plugins/cnc/  in the project, following this structure (one
connector per subdirectory).
Common Interface:  All plugins implement a common interface or abstract base class (for instance,
a class ProtocolPlugin  with defined methods). Typical methods include:
scan_network()  – to perform discovery and return a list of devices of that protocol.
connect(device)  – to establish a connection or session with a discovered device.
discover_tags(device)  – to enumerate all available telemetry points (registers, tags, variables)
on the connected device.
read_data(device, tags)  – to read or subscribe to the values of given tags (could be
implemented as an asynchronous loop or callback mechanism, depending on protocol).
Optionally, configure(device)  – any protocol-specific setup if needed (e.g. set sampling rate, or
in case of subscription-based protocols).
The main Mnemosyne core uses these interfaces to orchestrate the workflow. For example, after network
scanning,  for  each  detected  device  it  will  invoke  the  appropriate  plugin’s  discover_tags  to  get  all
telemetry points, then start continuous data collection via read_data  or equivalent.
Dynamic Plugin Loading:  Mnemosyne will automatically load all protocol plugins at runtime. This
could be done by scanning the  plugins  directory for modules or using a plugin registry. For
instance, Mnemosyne’s core might maintain a list/dictionary of available protocol classes keyed by
protocol name. New protocol modules can register themselves (e.g. via a decorator or an entry point
in setup.py  if packaged). The advantage is that adding a new protocol doesn’t require modifying
the core logic – the system will discover the new plugin and incorporate it. The plugin modules also
contain protocol-specific configuration (like default ports, scanning parameters, etc.) separate from
others.
Isolation and Dependencies:  Each plugin can use protocol-specific Python libraries. For example,
the Modbus plugin might use pymodbus  for implementing Modbus/TCP client; the OPC UA plugin
might use the python-opcua  library to handle sessions and browsing; the EtherNet/IP plugin could
use pycomm3  or cpppo  to talk CIP; the EtherCAT plugin uses pysoem ; the Profinet plugin might use
a custom DCP implementation or a wrapper to a C library if available. By isolating them, we ensure that dependencies for one protocol don’t interfere with others. This also means a plugin can be
disabled or removed if a certain protocol isn’t needed, to lighten resource usage.
Protocol Abstraction:  In the core logger , devices from different protocols will be handled uniformly
after discovery. For example, Mnemosyne can have a generic  Device object with attributes like
id,  protocol ,  address ,  name,  etc.,  and  a  reference  to  the  plugin  that  manages  it.  The
logging scheduler doesn’t need to know the low-level details of each protocol; it will call a standard
method (like device.read_all_tags() ) which under the hood calls the correct plugin’s logic for
that device. This abstraction makes the system extensible  – adding protocols doesn’t complicate the
main loop or database interface.
This plugin architecture ensures that Mnemosyne is modular and maintainable . Each protocol/asset type
lives in its own connector module, reflecting a separation of concerns (as noted, a “plugin architecture – each
asset type lives in its own connector package”  in the design). The architecture supports concurrent operation
of different protocols, meaning we can simultaneously monitor , say, a Modbus power meter and an OPC UA
robot controller with the same framework.

## Telemetry Autodiscovery for Discovered Devices
Once devices are found and identified, Mnemosyne proceeds to  autodiscover all available telemetry
(tags/registers/variables)  on each device. The goal is zero manual configuration: the system tries to learn
what data each device can provide. The strategy varies by protocol:
Modbus-TCP Devices:  Modbus itself does not have a standardized way to enumerate all registers or
coils on a device – you typically need a map from the manufacturer . However , Mnemosyne will make
a best effort. After identifying a Modbus device, the plugin can perform a probing sequence : Use the Report Slave ID  (Function 17) or Read Device Identification  (Function 43/14) to get device
details and perhaps a basic descriptor of supported features . Some devices include a “object list”
in the device identification, though this is uncommon.
Probe well-known register ranges based on device type. For example, many Modbus PLCs or
controllers have holding registers in certain spans. The plugin might do a quick read of small blocks
(e.g. 10 registers at a time) across the address space to see which addresses respond without error .
This must be done carefully to avoid flooding the device or writing to it accidentally. The result could
be a set of “active” addresses.
In practice, full autonomous discovery of Modbus registers is limited. Therefore, Mnemosyne will log
any known standard registers by default (e.g. if it’s a PowerTag meter by Schneider , log its
documented measurement registers). The design can also allow user-specified register maps  to
augment discovery for Modbus devices (as a fallback if auto-discovery is insufficient).
For each Modbus register or coil discovered/interested, Mnemosyne will create an entry in the tag list (with
address, type, etc.). These will then be read cyclically.
OPC UA Devices:  OPC UA is designed for rich self-discovery of data. For any OPC UA server , the
Mnemosyne OPC UA plugin will:
Connect to the server endpoint and browse the address space . Starting from the Objects folder , it
will recursively browse all nodes to find Variable  nodes (data points) available for reading. OPC UA’s
standard references allow the client to navigate the hierarchy of objects and variables.
As it discovers variables, it records their NodeId, browse/name, data type, etc. Many OPC UA servers
also provide metadata like Engineering Units, range, descriptions, which can be captured as part of
the tag metadata.
The plugin may apply filters to ignore certain administrative nodes or to focus on process data (for
example, ignoring diagnostics nodes unless requested).
OPC UA significantly eases telemetry discovery because the server itself exposes a model of all its data.
Mnemosyne effectively performs an  OPC UA tag crawl  to build the list of all available telemetry. This is
done once at connect time (and perhaps periodically or on subscription to catch newly added nodes). After
this, the plugin can create a subscription to all or critical variables, or it can poll them at a high rate
(depending on how the server is best used). The outcome is a complete list of tags for each OPC UA device.
EtherNet/IP (CIP) Devices:  CIP devices, especially PLCs, often support querying their data
definitions. For Rockwell Logix-series PLCs, Mnemosyne can use the List Tags  service (supported via
the Logix CIP services) to pull the entire tag list out of the PLC’s program . Using something like 
pycomm3.LogixDriver , one can call a method to upload the tag list  from the controller ,
which returns all user-created tags (with names, types, and possibly current values). Mnemosyne’s
CIP plugin will do exactly this for any Allen-Bradley/Logix PLC discovered. This yields a
comprehensive set of telemetry points (all PLC variables marked as accessible).
For other CIP devices (like an I/O block, drive, or sensor that speaks EtherNet/IP), there might not be a “tag
list” in the same sense. Instead, such devices often organize data into CIP  assembly objects  or specific
class/instance/attribute collections. Mnemosyne will handle two scenarios: - If the device supports CIP
UCMM (unconnected messaging), the plugin can perform generic CIP requests. It could query the Identity Object (class 0x01) to confirm device info (done in discovery) and the Assembly Object (class 0x04) to see
what cyclic I/O assemblies it has. Many devices have assembly instances for input/output data. If found,
Mnemosyne will treat those as telemetry (e.g. “Assembly 100 = status data structure”). - In cases where an Electronic Data Sheet (EDS) file is available (which lists device’s data items), Mnemosyne could parse that
(though implementing EDS parsing is complex and beyond initial scope). Instead, the plugin might just
collect any standard objects (like parameters, identity, assembly, perhaps vendor-specific objects if known).
In summary, for PLCs – get the tag database; for simpler devices – log their I/O assembly data and any
standard metrics. Each identified data item becomes a tag entry for logging.
EtherCAT Devices:  EtherCAT slaves typically follow the CANopen over EtherCAT (CoE)  standard for
configuration and telemetry. Once the EtherCAT plugin identifies the slaves on the bus (with 
master.slaves  list), it can leverage SDO (Service Data Object) reads  to get each device’s object
dictionary. The PySOEM library, for instance, allows reading CoE object entries by index/subindex
. Mnemosyne will:
Read the Identity Object (0x1018) to get basic info (vendor ID, product code, revision).
Iterate through standard object indices (0x6000-0x9FFF range for manufacturer-specific and
standard application objects, as well as any known standardized objects).
It can use the device’s ESI (EtherCAT Slave Information) if available; however , without an ESI, a
generic approach is to try reading all objects that the device reports as existing. PySOEM’s 
config_init  actually reads some basic dictionaries. We can also call master.config_map()  to
configure all process data.
The high-level approach: the EtherCAT plugin will configure the network for process data exchange  (PDO)
which gives live data from the slaves. It will also retrieve names of signals  if possible (some EtherCAT slaves
support an object that lists the names of their PDO entries). If not, it may generate generic names (like
“Slave1_TxPDO1_Channel1” etc.). All these discovered data points for each EtherCAT slave are then logged.
Essentially, Mnemosyne acts as a simple EtherCAT master that makes all slave I/O data available for logging.
PROFINET Devices:  Fully autoloading all telemetry from a PROFINET device is challenging, as it
typically requires the engineering configuration (GSDML file). However , Mnemosyne’s PROFINET
plugin will do the following:
After discovering the device with DCP, attempt to establish a PROFINET IO Controller connection (if
we have the capability in software) to read the device’s Module and Submodule list . PROFINET
devices are structured in slots and subslots (akin to modules). There are PROFINET services to read
the Identification & Maintenance records (I&M data) and the module directory. The OPC Foundation
reference suggests scanning with DCP Identify, then using PROFINET read services to get all asset
information continuously . We adopt a similar idea: send a Read Record request for known record
indices that contain data or I&M info.
If a full IO connection is feasible (perhaps using an open library or a minimal PNIO controller stack),
the plugin could actively pull input/output data cyclically. However , implementing a full PROFINET
controller in Python is complex. Instead, a simpler method is to use the device’s SNMP or web API if
available to gather telemetry (some PROFINET devices expose data via SNMP MIBs, etc.). This might
be outside the core scope.
Initially, Mnemosyne will focus on identifying the device and logging whatever standard data it can. For
example, many PROFINET devices are PLCs (Siemens S7 etc.), which might be better accessed via an S7
protocol  (Snap7  library)  for  actual  data.  (Future  plugin  possibility:  an  S7  protocol  plugin).  For  now, Mnemosyne’s PROFINET module will capture device identity and basic real-time data if possible (perhaps by
reading cyclic data records).
In all cases, once the plugin gathers the list of available telemetry points for a device, it will create entries in
the Mnemosyne database schema (in the tags/points table, see next section) and prepare for logging. The
telemetry discovery process is automatic  but also  safe  – the plugins will avoid writing to devices or
performing intrusive operations. They use read-only queries and standard discovery functions. If a device
requires authentication (e.g. some OPC UA servers, or PLCs with protected access), Mnemosyne will log that
the device is found but may require credentials (which a developer can provide via config to allow tag
browsing).

## High-Frequency Asynchronous Data Logging
Mnemosyne is built to log data at  high frequencies asynchronously , ensuring that it captures as much
detail as possible ("highest safe bandwidth") without overwhelming the devices or network. Key design
choices for achieving this include:
Asynchronous I/O and Concurrency:  Data collection from each device is handled in a non-blocking
manner . Each plugin module runs its own collection loop, either on separate threads or using 
asyncio  coroutines, so that waiting on one device’s response doesn’t stall others. For example:
The Modbus plugin might spawn a thread per device that continuously issues read requests (using a
small sleep or rate limit to match safe bandwidth).
The OPC UA plugin will likely use the subscription callback mechanism of the OPC UA library (where
the server pushes updates at a set interval), handled in an async event loop.
The CIP plugin can use threads or async if supported, polling multiple tags or using multiple
concurrent connections (Allen-Bradley PLCs allow multiple parallel requests, which pycomm3  can
manage).
EtherCAT reading will be tied to the cycle of the bus; typically a loop running at, say, 10ms (100 Hz) or
1ms (1 kHz) if the system can handle it. We will use a high-priority thread for EtherCAT that reads the
process data and then yields to other tasks.
Profinet, if implemented via a simple approach, might also poll at some frequency.
Python’s GIL might impose some limitation on pure Python threads at extremely high rates, but since much
of the I/O will be waiting on network, the GIL is less a bottleneck. If needed, critical loops (like EtherCAT)
could use a C extension or the fact that pysoem releases GIL during I/O.
Highest Safe Bandwidth:  Mnemosyne will aim to log as fast as each device safely  allows. “Safe”
means not saturating the device’s CPU or the network to the point of affecting normal operations.
The default polling interval or subscription rate might be set to, say, 100ms (10 Hz) for all tags, but if
a device supports faster and the network is reliable, it could increase. For instance, if an EtherCAT
device can be polled at 1 kHz (like a servo drive position), we attempt that. OPC UA subscriptions can
often be set to very low publishing intervals (like 50 ms or less) – we would configure the fastest
update rate that the server permits for each monitored item.
The system might include an  adaptive rate control : monitoring the responsiveness of each device. If
timeouts or delays occur , Mnemosyne can back off polling frequency for that device to prevent overload.
This is essentially a back-pressure  mechanism – if the data pipeline (or the device) is getting bogged down,
the effective polling rate is reduced to a sustainable level. This concept of back-pressure control ensures
high-rate I/O without data loss.
Parallel Data Capture:  Since multiple devices and protocols are handled concurrently, Mnemosyne
can in total handle a large throughput. For example, reading 100 tags at 100ms interval from one
PLC, 50 tags at 500ms from another device, etc., all interleaved. The design avoids having one slow
device block others. It uses either multi-threading or asynchronous multiplexing to schedule reads.
In Python, libraries like asyncio  can be leveraged (especially for OPC UA which often has an async
API, or any socket I/O). For simplicity, a combination of thread pools (for blocking I/O like Modbus
TCP) and async for those that support it can be used.
Buffering and Queueing:  Collected data points from each plugin are placed into a thread-safe
queue or buffer that feeds the database writer . This decouples data acquisition from disk/database
writes. Each read cycle from a device creates a timestamped data record (or batch of records) and
enqueues it. A dedicated logging thread  (or async task) pulls from this queue and writes to the SQL
database. If the database is slow or temporarily unavailable, the queue will backlog (up to a certain
limit), signaling the acquisition threads to slow down (back-pressure). This way, momentary slowness
in writing (or spikes in data volume) don’t force the acquisition to drop data, unless the queue fills
beyond capacity – in which case the system can start dropping the oldest data or skip some samples
to recover .
Timestamping:  Each data sample is timestamped with a high-resolution time source on the laptop
(e.g. using Python’s  time.time()  or  datetime.now()  in UTC). If a device provides its own timestamp (some OPC UA variables might, or a PLC might have a system clock), those can be logged
as separate fields, but the primary timeline will be the collection time on the logger . This provides a
unified chronology across all devices. Using the laptop’s NTP-synced clock ensures consistency for
integration with other systems (like the digital twin).
Data Volume Considerations:  High-frequency logging can generate a lot of data. Mnemosyne’s
design anticipates this and uses efficient writing (discussed below in the database section) and
possibly data compression. For example, if logging at kHz across many tags, the system may choose
to batch multiple samples together for insertion to reduce overhead. Also, the configuration may
allow a maximum rate or enable only certain tags for high-frequency logging (less critical tags at
slower rate). The developer can adjust these in the CLI/config (e.g. specifying a global max poll rate,
or per-device rates).
In summary, Mnemosyne’s data collection engine is built to be  asynchronous, high-throughput, and
robust . It can comfortably achieve sub-second sampling intervals on capable networks, and it safeguards
against overload through buffered asynchronous processing. This ensures that the digital twin has access
to rich, high-resolution telemetry data for analysis and that no single slow component will derail the overall
logging process.

## SQL Database Schema Design and Setup for DBeaver
All telemetry is logged into a  structured SQL database  (such as PostgreSQL or SQLite). The schema is
designed  for  efficient  retrieval,  especially  time-series  queries  per  device  or  sensor ,  and  for  easy
management (the user can open it in DBeaver or any SQL client to inspect and query the data). Below is the
proposed schema and organization:
Devices Table ( devices ): This table catalogs all discovered devices. Each entry includes:
device_id  (primary key) – unique identifier (could be an auto-increment integer).
protocol  – the protocol/plugin (e.g. "Modbus", "OPC-UA", "EtherNetIP", "EtherCAT", "Profinet").
address  – network address or identifier (e.g. IP address for IP-based devices, or MAC for those
without IP like raw EtherCAT, or COM port if serial in future).
name – human-readable name of the device (if available, e.g. from OPC UA server name, CIP
identity object, PROFINET device name).
type – device type or model (if known, e.g. "Siemens S7-1500 PLC", "Allen-Bradley ControlLogix",
"Schneider PowerTagMeter").
Additional metadata: vendor, firmware_version , etc., if easily obtained during discovery
(these can be columns or a JSON blob for flexibility). For example, CIP ListIdentity returns vendor and
device info which we can store .
This table allows quick lookup of what devices are being logged and their key info. It’s useful for filtering
queries by device or joining to get device attributes.
Tags/Points Table ( tags or telemetry_points ): This table lists all telemetry points (sensors,
registers, variables) that are being logged from all devices.
tag_id (primary key).
device_id  (foreign key to devices ).
tag_name  – name of the telemetry point. For OPC UA, this could be the full browse path or a
variable name; for PLC/CIP, the tag name in the PLC program; for Modbus, perhaps an address or a
label like "40001" or "Pressure"; for EtherCAT, a combination like "Slave1.Chan0".
address  – an address or identifier in the device context. E.g., a Modbus register number , an OPC
UA NodeId, a CIP instance path, etc. This can be stored as text for human reference.
data_type  – e.g. INT, FLOAT, BOOLEAN, STRING, etc., indicating how to interpret the value.
units – if available (OPC UA and some others provide engineering units).
description  – longer description if available (some protocols might provide descriptions).
This table essentially is a data dictionary  of all collected signals. It’s useful for metadata queries (like listing
all tags of a device) and for the digital twin to understand what each point is.
Telemetry Data Table ( telemetry ): This is the core time-series table that stores the values over
time.
It can be structured in a few ways; the simplest is a wide schema : each row = one measurement of
one tag at a certain time:
tag_id (FK to tags table).
device_id  (to avoid join, could be redundant since tag->device, but including for
convenience and indexing).
timestamp  – date-time of the measurement (likely with timezone or as UTC epoch). This
should be precise (e.g. up to milliseconds or microseconds if needed).
value – the value of the telemetry point at that timestamp. This could be a numeric type (if
most values are numeric) or text. We can use SQL’s flexible types: e.g. a numeric (double
precision) column for numeric readings and perhaps a separate text column if needed for
non-numeric. Alternatively, store everything as text (string) and parse as needed, but that’s
less efficient for numeric analysis. A pragmatic approach: have separate columns for each
base type and only one will be filled (e.g. value_num , value_text , value_bool )
depending on the data type. However , to keep it simpler and more normalized, using a single 
value column of type TEXT and another data_type  to interpret can work, albeit with
some performance cost for numeric queries.
Optionally, quality  or status – some protocols provide quality status (OPC UA has
Good/Bad status for each reading). Could include that.
This  telemetry  table  will  be  indexed  on  (tag_id,  timestamp)  and/or  (device_id,  timestamp)  for  efficient
querying of time series. With such indexes, one can quickly retrieve all data points for a given tag or device
in a time range (which is the common query for analysis or visualization).
If using PostgreSQL, we could also leverage PARTITIONING  by time (e.g. one partition per day or month) to
keep the table manageable for very large datasets. Or use a time-series extension like TimescaleDB on top
of Postgres for automatic chunking and fast aggregation. This is optional but worth noting for scaling.
Schema Example:  For illustration, in PostgreSQL syntax:
CREATETABLEdevices (
device_id SERIALPRIMARY KEY,
protocol VARCHAR(20),
address VARCHAR(100),
nameVARCHAR(100),
typeVARCHAR(100),
vendorVARCHAR(100),
modelVARCHAR(100),
firmware VARCHAR(50)
-- etc.
);
CREATETABLEtags(
tag_idSERIALPRIMARY KEY,
device_id INTREFERENCES devices(device_id ),
tag_name TEXT,
address TEXT,
data_type VARCHAR(20),
unitsVARCHAR(20),
description TEXT
);
CREATETABLEtelemetry (
tag_idINTREFERENCES tags(tag_id),
device_id INT,
timestamp TIMESTAMP WITHTIMEZONE,
valueTEXT,
PRIMARY KEY(tag_id,timestamp )
);
CREATEINDEXidx_telemetry_device_time ONtelemetry (device_id ,timestamp );
(This is a simplified schema; actual implementation might refine data types and constraints.)
Usage with DBeaver:  Both SQLite and PostgreSQL are easily accessible in DBeaver:
If using SQLite , Mnemosyne could store data in a local file mnemosyne_data.db . After running the
logger , a developer can open this file in DBeaver (just establish a new SQLite connection to the file)
and run SQL queries or use the UI to explore tables. SQLite is convenient for a single-user scenario
and zero setup.
If using  PostgreSQL , one would need a running Postgres server (could be local or on another
machine). Mnemosyne would be configured with the connection URL (which can be provided via CLI).
Once data is logged, the developer uses DBeaver to connect to that same database (provide host,
port, credentials in DBeaver). The schema design ensures the data is in clear relational tables, which
DBeaver  can  display.  The  user  can  execute  complex  SQL  (joins,  aggregates,  etc.)  to  analyze
telemetry, or export data to CSV, etc., all through DBeaver .
Efficiency and Maintenance:  The structured schema makes it straightforward to perform queries
like:
“Give me all temperature sensor readings from Machine X between two timestamps” (join devices
and tags to telemetry, filter by name and time).
“Show latest value of each tag for each device” (a grouping or MAX timestamp query).
Since the data is time-series heavy, we anticipate the telemetry table growing large. Using indexes
and possibly partitioning will keep queries fast. The design also allows archiving  old data: e.g., older
partitions could be compressed or moved out after certain retention period, to keep the working set
small (these details could be part of a maintenance guide).
Integration with Digital Twin/AI:  Storing telemetry in SQL makes it accessible for OrionFlow’s
digital twin or AI modules. For RAG (Retrieval Augmented Generation) workflows, one could easily
query specific slices of data to feed into an AI model prompt. The schema could be leveraged to
quickly retrieve relevant historical data (since it’s organized by device and tag, an AI agent could find,
say, all anomalies for a specific sensor by querying this DB, etc.). The important point is that the data
is in a  standard relational form  with clear relationships, which any tool (including DBeaver or
Python scripts or AI agents) can use without proprietary formats.
Mnemosyne will include a database setup guide : e.g., instructions to install Postgres if needed, the DDL to
create the schema (which the program can also auto-create on first run), and examples of connecting via
DBeaver . For instance, the README might have a step: "Open DBeaver , create a new connection to SQLite
mnemosyne_data.db  or  to  Postgres  (host:  localhost,  db:  mnemosyne,  user/password),  then  you  can
browse tables devices , tags, telemetry  to see your logged data." This helps ensure that developers
and engineers can quickly validate and utilize the collected telemetry.

## Command-Line Interface and Configuration
Mnemosyne provides a  command-line interface (CLI)  that allows developers to configure and run the
telemetry logger easily. The CLI is designed to be intuitive for developers who may want to adjust scanning
parameters, database settings, or logging options. Key CLI features include:
Selecting  Network/Subnet:  You  can  specify  which  network  interface  or  IP  range  to  scan.  For
example: 
mnemosyne --subnet 192.168.1.0/24
This would limit scanning to that subnet. If not provided, Mnemosyne might default to the local
machine’s primary network. Alternatively, a flag --iface eth0  could be used to specify an
interface for EtherCAT or PROFINET scanning (since they might use raw Ethernet).
Database Configuration:  Flags to choose the database type and location:
--db can accept a connection string or type. E.g. --db sqlite:///path/to/data.db  for
SQLite, or --db postgresql://user:pass@host:port/dbname  for Postgres. 
Other options like --db-user , --db-password , --db-host  may be provided, but a single URL
is often convenient.
If not provided, defaults could be used (like a local mnemosyne.db  SQLite file).
Protocol Enable/Disable:  By default, Mnemosyne will attempt all protocols it supports. But the CLI
can allow enabling or disabling specific plugins. For instance: 
mnemosyne --enable OPC-UA,Modbus --disable EtherCAT,Profinet
This would only scan for OPC UA and Modbus devices, ignoring EtherCAT/Profinet (useful if you
know your environment, to speed up scanning or avoid unnecessary logs). If not specified, all are
enabled.
Logging Level and Output:  Standard verbosity flags:
-v or --verbose  to print more detailed logs to console (e.g. each discovery, each value
occasionally).
--quiet  to run with minimal output (just errors).
Logging can be directed to a file via --log-file mnemosyne.log  if needed, which is helpful for
debugging long runs.
Rate and Bandwidth Settings:  Developers can tweak performance with CLI options:
--poll-interval 0.1  for example to set a global poll interval of 0.1 seconds (10 Hz) if we want
to throttle globally.
Or --device-poll OPCUA:0.05,Modbus:0.5  to specify different base rates per protocol.
Possibly --no-backpressure  to disable adaptive slowing (for testing maximum throughput).
These options help adapt Mnemosyne to different scenarios (for instance, in a testing environment you
might push to max frequency; in production you might be conservative).
Output Data Format Options:  By default, data goes to the SQL database. But CLI might offer
additional outputs for convenience:
--csv to also write updates to a CSV file (mainly for quick tests or if someone doesn’t want to use
DB).
--parquet  to export hourly Parquet files (if one wants columnar storage in addition, although the
main ask is SQL, this could be an optional feature influenced by earlier design considerations).
--no-db  to run a discovery only (not logging data, just listing devices and tags). This is a useful
mode to see what would be logged without actually storing data.
Configuration File:  Instead of specifying many options via CLI each time, Mnemosyne can support a
YAML/INI configuration file: 
mnemosyne --config config.yaml
The config file can contain the same info (subnets, db, etc.), plus more complex settings (like
credentials for OPC UA servers, which tags to exclude if any, cloud integration settings, etc.). This separation is useful for developers who run this regularly; they can version-control the config file for
a given site.
Interactive/Diagnostic Commands:  The CLI could include subcommands for utility:
mnemosyne scan  – performs discovery and prints out a list of devices (with protocol, IP, name) and
perhaps summary of found tags per device, then exits. (No data logging).
mnemosyne log  – the default command that does scanning then continuous logging.
mnemosyne export --device X --start TIME --end TIME  – a possible command to export
logged data (from the SQL DB) to a CSV or other format for analysis outside. Although one can do it
in DBeaver or SQL, a CLI option is user-friendly for quick exports.
mnemosyne test  – could simulate devices or run self-tests (like connecting to a dummy server ,
useful for development).
Ease of Use:  The CLI’s help ( mnemosyne --help ) will document all options. We aim for sane
defaults so that running  mnemosyne  with no arguments will “just work” on the primary network
and log to a default local database. Developers can then refine via options as needed. 
In essence, the CLI and config capabilities ensure developers can quickly deploy Mnemosyne in different
environments  (point it at the correct network, choose where data goes, adjust performance) without
touching the code. This makes the tool flexible for various use cases – from a quick troubleshooting session
on a laptop to a continuous logging service in a plant.

## Extensibility for Future Protocols and Cloud Integration
One of the design goals is future-proofing: Mnemosyne should be easy to extend with new data sources
(additional protocols, including serial/fieldbus) and data sinks (like cloud storage or IoT platforms). We
address this in the design:
Adding New Protocol Plugins:  The plugin architecture is ready to accept new protocols beyond the
initial Ethernet-based ones. For example, serial or fieldbus protocols  such as PROFIBUS, CANopen,
Modbus-RTU, etc., can be integrated by writing new plugin modules.
Serial Protocols:  Since the core system runs on a laptop with a standard Ethernet port, direct serial
(RS-232/485) or fieldbus (like PROFIBUS DP which is RS-485 based) might require a USB adapter or
gateway. The design doesn’t assume those are present, but it doesn’t preclude them. A plugin for
Modbus-RTU could, for instance, use a USB-to-RS485 dongle and a library like pymodbus (with serial
support) to scan COM ports for devices. The architecture would allow specifying a COM port in config
for that plugin. Similarly, a PROFIBUS plugin might interface with a PROFIBUS-to-Ethernet proxy or a
PC card if available. While not plug-and-play on a vanilla laptop, the hooks are there to integrate
such data when needed.
CANopen:  Perhaps a CAN-to-USB adapter could be used. A CANopen plugin would scan the CAN
network for node IDs and use a library or socketCAN to read data. Mnemosyne’s modular design
would accommodate this by treating it like another plugin with its scanning and data reading
methods.
In all cases, adding a plugin would involve creating a new folder in plugins/ , implementing the
interface, and possibly adding any external library in requirements. Because of dynamic plugin
loading, the core doesn’t need changes. For example, a developer adding a plugins/profibus/
module can integrate it, and Mnemosyne will pick it up and run its scan method. The Contributing
Guide  for Mnemosyne will outline how to add connectors, making it straightforward for an engineer
to extend.
Cloud Export Hooks:  OrionFlow’s platform likely involves cloud or central database integration for
aggregated  analysis  across  many  sites.  Mnemosyne,  being  an  edge  logger ,  can  support  cloud
export  in a couple of ways:
Batch Uploads:  The system could periodically upload logged data (or live stream) to a cloud
endpoint. For example, an extension module could read from the local SQL database or subscribe to
the internal data queue and send data to a cloud database or message broker . This could be
implemented as an optional plugin or separate service that connects to Mnemosyne.
MQTT or REST Streaming:  As hinted by the design, Mnemosyne might include an MQTT connector .
This could mean two things: reading from MQTT (if devices publish telemetry via MQTT, treat it as
another protocol plugin), and/or publishing to MQTT (taking the logged data and publishing it to an
MQTT topic for cloud ingestion). The mention of MQTT in the context suggests considering MQTT as
a telemetry source or sink. With a plugin approach, an MQTT output plugin  can be added which
subscribes to the internal event bus of Mnemosyne and publishes each data point (or certain
summaries) to an MQTT broker (which could be cloud-hosted).
APIs and Integration:  We can provide integration points such as a callback or observer pattern:
whenever a new data point is logged, call any registered “exporters”. An exporter could be a simple
function that writes to an external API (like an HTTP POST to a cloud REST API) or pushes to a data
lake. By keeping this separate from the main logging thread, we ensure local logging is robust (for
instance, if network to cloud is down, local logging still continues; data can be retried for upload
later).
The database schema is also cloud-friendly. If OrionFlow’s digital twin runs in the cloud, one strategy
is to use Postgres on the cloud and have Mnemosyne log directly to that (assuming connectivity).
The CLI could be pointed to a cloud DB connection string. In remote factory scenarios, a local SQLite
might  collect  data  offline,  and  an  exporter  process  could  sync  it  to  a  cloud  database  when
connectivity is available.
Configuration for Extensions:  The design includes config placeholders for future features. For
example:
In the config file, a section for cloud upload
(cloud: enabled: true, mode: MQTT, broker: x, topic: y  or mode: HTTP, 
endpoint: ... ).
Options for data thinning or preprocessing (maybe in future we add capability to send only
aggregated data to cloud to save bandwidth).
Extensibility also covers custom data processing: e.g., a hook to run user-defined Python code on
each data point or each device’s data stream (could be used for local anomaly detection or custom
logging logic).
OrionFlow  Integration:  Since  OrionFlow  likely  has  its  own  cloud  AI  system,  Mnemosyne  could
evolve to connect directly to OrionFlow’s ingestion API. For now, the focus is on local logging, but the
modular design ensures that incorporating OrionFlow’s SDK or API would be done in one place (e.g.
in the exporter plugin) without modifying the core logger or the protocol plugins.
Finally, the emphasis on no dedicated hardware assumptions  remains even with extensions: any future
protocol added should ideally use generic interfaces (network sockets or common adapters). If a specialized
interface is needed (like a PCI card for PROFIBUS), the system should detect its absence and simply not
enable that plugin rather than fail overall.
In  conclusion,  Mnemosyne’s  design  is  robust  and  flexible:  it  discovers  devices  over  multiple  industrial
Ethernet protocols, modularly interfaces with each via plugins, automatically learns what data to collect,
and logs high-frequency telemetry into a well-structured SQL database. The data is organized for easy
retrieval, supporting digital twin visualization and AI-driven analytics. With a developer-friendly CLI and an
extensible architecture, Mnemosyne can be adapted to new protocols and integrated with cloud systems,
ensuring  that  as  OrionFlow’s  needs  grow  (additional  machines,  protocols,  or  cloud  connectivity),  the
telemetry logger can grow with it. The result is a comprehensive, scalable telemetry ingestion system
that turns factory floor data into a queryable resource for optimization and predictive maintenance. 

## Sources:
Modbus function for device identification ; structure returns manufacturer and product info
. 
OPC UA discovery via mDNS/SLP for automatic server detection . 
EtherNet/IP (CIP) device discovery and tag listing (Logix) . 
PROFINET discovery using DCP Identify broadcast  and reading asset info via PROFINET services
. 
EtherCAT scanning using PySOEM (SOEM) master to find slave devices . 
Reddit discussion of CIP discovery tool using pycomm3 . 
Modbus Protocol
https://www.modbustools.com/modbus.html
Function 43-14: Read Device Identification (Basic) - PowerTag Link User Guide
https://www.productinfo.schneider-electric.com/powertaglinkuserguide/powertag-link-user-guide/English/
BM_PowerTag%20Link%20D%20User%20Manual_4af62430_T000501355.xml/$/TPC_Function4314_4af62430_T000501597
Module pyModbusTCP.client
https://pymodbustcp.readthedocs.io/en/latest/package/class_ModbusClient.html
OPC UA and the OPC Discovery - Open Platform Communications
https://www.opc7.com/opc-ua-and-opc-discovery/
Python code and EXE to discover all CIP devices on a network : r/PLC
https://www.reddit.com/r/PLC/comments/1hh4ih4/python_code_and_exe_to_discover_all_cip_devices/
pycomm3 1.2.14 documentation
https://docs.pycomm3.dev/en/latest/
Welcome to PySOEM’s documentation! — PySOEM 1.1.12 documentation
https://pysoem.readthedocs.io/en/latest/
GitHub - Eiwanger/nmapProfinet: A nmap script to discover profinet devices in a lokal subnet with the
pn_dcp call
https://github.com/Eiwanger/nmapProfinet
PROFINET - 5.3.2.5 Asset discovery
https://reference.opcfoundation.org/PROFINET/v101/docs/5.3.2.510
