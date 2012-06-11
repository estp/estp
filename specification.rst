===========================================
Extensible Statistics Transmission Protocol
===========================================

:Author: Paul Colomiets <paul@colomiets.name>
:State: draft
:Version: 0.3


Abstract
========

There are various monitoring and statistics collection solutions.  Most of them
are not interoperable.  Occasionally there is also the need to implement custom
processing for statistics data.  The goal of this protocol is to have a common
protocol for building statistics processing solutions.


Scope of the Protocol
=====================


Protocol Works on Top of Existing Framing
-----------------------------------------

The original intention was to have protocol working on top of ZeroMQ and
Crossroads IO, but the only real requirement for the underlying protocol is
framing (or message delimitation). So UDP, SCTP, and Websockets can also be
used as underlying protocols.

We also do not provide a way to group messages into batches. The Nagle
algorithm does this for TCP. For ZeroMQ and Crossroads IO a special device
can be implemented that transparently compresses and decompresses a bunch of
messages.


Interval Based Accounting
-------------------------

The protocol is suited to sending statistical counters reported at short regular
time intervals, e.g. CPU usage or messages passed though a gateway. It is not
suited to sending massive number of statistics (e.g. on each user request or each
network packet). It is not suited to transmitting large chunks of data at large
intervals.


Easy Filtering
--------------

The protocol is designed to be filtered by subscriptions implemented in ZeroMQ
and Crossroads IO. This lets us subscribe only to subsets of statistics data.
This gives a limitation of one value per message.


No Authentication, Compression and Encryption
---------------------------------------------

From the point of view of the authors of the protocol, the authentication,
compression, and encryption are better done by underlying protocols. There
are protocols for that. E.g. SSH or one of the various VPN implementations.


Overview
========

A typical packet looks like this::

    ESTP:org.example:sys::cpu: 2012-06-02T09:36:45 10         7.2
         ^ host      ^ name    ^ timestamp         ^ interval ^ value

The hostname is in reverse domain notation. After the last colon, whitespace is
insignificant, but at least one space should separate fields. The value can
have a type marker which is described below.

Another example that uses resource name and counter type (see below)::

    ESTP:org.example:network:eth0:bytes_written: 2012-06-02T09:36:45 10 1000000^

And example that shows that more hierarchy can be introduced in metric
names, if required::

    ESTP:org.example.s1:disk.usage:system/root:free.sectors: 2012-06-02T09:36:45 3600 123456789

Everything after the line-feed (LF) character (if present) is treated as
extension data. Each line of extension data must start with at least one SPACE
character.  Lines starting with exactly one space and a colon (" :") are
reserved for official extensions. Protocols built this way looks nice when a
stream of data is printed as-is.

Here is an example of the packet with collectd extension::

    ESTP:org.example:sys::cpu: 2012-06-02T09:36:45 12.3 10
     :collectd: type=cpu

It sets collectd's type for the value to something more specific for collectd
than the generic floating point type.


Protocol Structure
==================


Basic Structure
---------------

The protocol is text-based. Only printable ASCII characters are allowed.
However, the parsers are allowed to not to check this constraint.

The first line of the data frame is the metric itself. The line consists of
the following whitespace-separated parts:

* metric full name
* timestamp
* measure interval
* value and value type marker

Each part is described in detail below.

The following lines of the data frame are extension data. Each line of
extension data should be started by at least one space. Otherwise extension
data lines can contain arbitrary data. Official extensions are started with
space and colon, and last until the end of the line. All official extensions
must be concentrated, following the metric line. All unofficial extensions must
follow official ones.


Metric Full Name
----------------

Metric full name is a first part of the data frame. It is a string which starts
with "ESTP:" and ends with a colon ":". The string consists of 4 parts
separated by a colon. Each part can contain arbirary characters except
whitespace and a colon (note only printable ASCII characters are allowed in the
whole data frame).

The parts of the metric name are following:

1. Hostname of the origin server in reverse domain notation
2. Application name
3. Resource name
4. Metric name

The concatenation of them form the prefix which you can subscribe to using
zeromq or libxs. The final colon is needed to be able to subscribe to the exact
metric. If it was absent you could either subscribe up to a metric name (e.g.
``ESTP:org.example:sys::cpu``) which would match the arbitrary suffixes (like
``ESTP:org.example:sys::cpu2``), or need to subscribe with each whitespace at
the end (at least space and tab). There is also a opportunity to subscribe
to a specific host, specific application and specific resource or a prefix
of any of them.

Application name, resource name and metric name can also contain hierarchy
inside. The convention is to use dot as the hierarchy delimiter. However, using
a native way to specify hierarchy is preferred if exists (e.g. using ``/`` if
the name is a file path of part of it)

Each part is limited for up to 63 bytes. Giving 261 byte limit for full name.


Hostname
````````

The hostname is one of the following:

* A fully qualified domain name in reverse notation. That means that top level
  domain goes first, second level next, and so on (e.g. ``org.example.server``)

* IPv4 address in most commonly used dotted decimal notation (e.g.
  ``127.0.0.1``)

* IPv6 address in lowercased hexadecimal format, omitting colons and without
  abbreviation (e.g. loobback ``::1`` address is
  ``0000000000000000000000000000000001``). This ensures that there is only one
  possible representation of the given ip address or network prefix.

Note: no reversing is applied to ip addresses.

These rules are invented to give a simple and obvious way to subscribe to the
subnetwork (or whatever is represented in domain name hiererarchy). Whenever
possible, domain names are preferred over ip addresses, as they are more human
readable and are not influenced by low level networking changes.


Application Name
````````````````

This is a name of the application which submits statistics data. When using
full featured statistics server it can be subsystem name.

The application name is expected to denote subset of statistics which is common
for this application (or subsystem) across different hosts.

The empty application name is reserved for future versions of this protocol (a
set of well-known application-independent metrics).

Examples:

* ``disk``
* ``ping``
* ``ntpd``
* ``HDFS.NameNode``, ``HDFS.DataNode``


Resource Name
`````````````

The resource name is a name of the resource local to  the application or
subsystem. It can be empty for simple applications.

Different resources across single application (or subsystem) are expected to
have identical metrics.

Examples:

* ``sda1``, ``system/root`` (disk sybsystem, latter is lvm partition)
* ``eth0`` (network subsystem)


Metric Name
```````````

The final component is a metric name.

Examples:

* ``cpu_time``
* ``sent.packets``, ``sent.bytes``


Timestamp
---------

Timestamp is ISO8601 combined date and time in extended format with the second
precision. Date and time is always an UTC, to avoid ambiguity.

Examples:

* ``2012-06-06T14:54:12``

Note: ISO8601 basic format (without dash and colon signs) is not supported.
Format support may be extended in the future revisions of the protocol to
allow better precision of timestamps.


Measure Interval
----------------

Measure interval is just a number of seconds that is expected between
subsequent reports for this specific metric.

Note: The field may be extended to support higher precision measure
interval in the future revisions of the protocol by using decimal point


Value and Type
--------------

The simplest value consists of digits and decimal dot if needed. It's either
integer or double-precision floating point value. This kind of value is called
"gauge" in RRD and Collectd.

Other value types are denoted by appearance of colon and data type letter
after colon:

* ``c`` -- a ever-growing "counter"

* ``d`` -- a "derive" type

* ``a`` -- a "delta" type

* ``x`` -- an extension type, the real type must follow ``x`` character

Examples:

* ``10``, ``45.123``
* ``123456789:c``
* ``2345.234:d``
* ``123:a``
* ``1ab4:x-my-type``

Note: Only first character after colon denotes type, all following characters
must be ignored (they are reserved for optional parameters of the type). The
types starting with "x" are free to use in any application for experimenting,
at risk of incompatibily problems between implementations.

Counter Type
````````````

The counters which accumulate number of events can be sent as is using this
type. The example of such counters are number of bytes sent throught the
gateway, or number of email messages sent by mail agent.

Note: This version of the protocol has no notion of counter wrap. This
event is deemed as insignificant, or at least less important than treating
counter reset as counter wrap. So for RRD-based implementations its
recommented to use DERIVE type with minimum of 0 to store value of ESTP
counter type


Derive Type
```````````

This type is basically similar to counter except the number is expected to be
able to become lower. For example if you have a free space
on hard drive, you may want to know how the data size changes over time.

Note: We have no maximum and minimum limits at the moment. So it's
impossible to find out whether the value was just reset or the delta is so
big. So use of this type is not recommended for non-persistent counters.


Delta Type
``````````

Number with this type denotes the number of events occured during the interval.

Note: This is ABSOLUTE type from the RRDTool, but I find that name
misleading


Choosing the Right Data Type
````````````````````````````

Selecting the data type is trivial problem although not very obvious at first
glance.  There are some rules of thumb:

1. Use gauge and delta type if possible as it allows stateless inspection of
   statistics

2. Use gauge on values that are same for any measuring interval (e.g. CPU usage
   or memory free)

3. Use delta type for events over time values. It's not only semantially right,
   it also allows to change interval without loosing old data, and it allows GUI
   to display rate values over larger periods of time (e.g. you submit messages
   per ten second period, and GUI shows messages per hour and messages per day)

4. Use counter type when you have no state, or when you have no control over
   when exactly the value is measured (or in other words if your timer is
   inacurate). E.g. getting received bytes from router by SNMP may take a time if
   network load is high.

5. Use derive type when you need to track the change of some value. Do not use
   it for volatile counters which can be reset on software restart, and on
   counters that wrap. Example of such value is database size: it's fully
   non-volatile value and it's grows is more interesting that the full size.


Forward Compatiblity
====================

This section defines the extension points where protocol can be improved in the
future. The decision whether to implement the rules outlined here is given to
the protocol implementor, but we highly encourage to consider them to build
interoperable and future proof protocol implementation.

The following are extension points free for any use:

* The extension section may contain arbitrary data (except limits specified
  above) that suits need of particular project or application. For easier
  parsing it's recommended that each extension's data is contained in single
  line

* Types starting with "x". Before ``:x`` any data is allowed except colon and
  reserved characters (``:,;``). After the type, any characters are allowed
  (note, that no whitespace is allowed in the value field).  However, using
  such types is highly discouraged for interoperability reasons

There are the following provisions for future revisions of the protocol:

* Timestamp field may be extended to support better precision. Parser should
  skip trailing characters after the timestamp for forward compatibility

* Annotations may be added to the value field (e.g. measure units). For this
  case ``,`` (comma) and ``;`` (semicolon) characters are reserved. Forward
  compatible parser should ignore everyting after the reserved characters, the
  ignored data is guaranteed to have no influence on semantics of the value

* More types may be added, if the type character is not in the documented set
  of types (``c``, ``d``, ``a``), the value must be treated as undefined
  (including ``x`` character, unless that type is specified by
  application-defined means)

* Existing types may be parametrized. The forward compatible parser should
  ignore everyting after type marker. The parser must check only first
  character of the type. Gauge type is guaranteed to be never parametrized
  (however, the annotations may be added as defined above)

* Additional fields may be introduced after the value field

* As the last resort ``ESTP:`` prefix will be changed to ``ESTP2:`` if
  incompatible changes would be introduced


Copyright
=========

This document has been placed in the public domain.

For the legal description of the statement above see:

http://creativecommons.org/publicdomain/zero/1.0/
