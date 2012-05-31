===========================================
Extensible Statistics Transmission Protocol
===========================================

:Author: Paul Colomiets <paul@colomiets.name>
:State: draft
:Version: 0.1


Abstract
========

There are various monitoring and statistics collection solutions.  Most of them
are not interoperable.  Occasionally there is also need to implement custom
processing for statistics data.  The goal for this protocol is to have a common
protocol to build statistics processing solutions.


Scope of the Protocol
=====================


Protocol Works on Top of Existing Framing
-----------------------------------------

The original intention was to have protocol working on top of Zeromq and
Crossroads IO, but the only real requirement to the underlying protocol is
framing (or message delimitation). So UDP and SCTP can be used as underlying
protocol too, as well as Websocket.

We also do not provide a way to group messages into bulks. There is Nagle
algorithm for this task in TCP. For Zeromq and Crossroads IO a special device
can be implemented that transparently compresses and decompresses a bunch of
messages.


Interval Based Accounting
-------------------------

The protocol is suited to send statistical counters reported at short regular
time intervals , e.g. CPU usage or messages passed thought gateway. It is not
suited to send massive number of statistics (e.g. on each user request or each
network packet). It is not suited to transmit large chunks of data at big
intervals.


Easy Filtering
--------------

The protocol is designed to be filtered by subscriptions implemented in Zeromq
and Crossroads IO. This allows to subscribe only to subset of statistics data.
This gives a limitation to have only one value per message.


No Authentication, Compression and Encryption
---------------------------------------------

From the point of view of the authors of the protocol, the authentication,
compression and encryption are better done by underlying protocol. There are
plenty of protocols for that. E.g. SSH or one of the various VPN
implementations.


Overview
========

Typical packet looks like following::

    ESTP:example.org:system::cpu:     1338277275   7        sint64   gauge
         ^ host      ^ name           ^ timestamp  ^ value  ^ type1  ^ type2

After the last colon, whitespace is insignificant, but at least one space
should separate fields. The ``type1`` is storage type for the value, and
``type2`` is data source type, similar to one used in round-robin databases
(RRD).

Everything after line-feed (LF) character (if exists) is treated as extension
data. And each line of extension data must start with at least one SPACE
character.  Lines starting with exactly one space and a colon (" :") are
reserved for the official extensions. Protocol built this way looks nice when
stream of data is printed as is.

Here is an example of the packet with collectd extension::

    ESTP:example.org:system::cpu: 1338321583 12.3 double gauge
     :collectd: type=cpu

It sets collectd's type for the value to something more specific for collectd
than the generic floating point type.


Protocol Specification
======================

TODO


Example Parsers
===============

The following examples are here to show how simple parser is needed for the
protocol. We omit some error checking, which is crucial for real production
code. But the parser we show here is good to parse any valid input and
is forward compatible.


Example in C
------------

Protocol is parsed easily with ``sscanf``. The following is brief example::

    end = strchr(source_message, '\n');
    if(end) *end = '\0';
    sscanf(source_message, 'ESTP:%[^:]:%[^:]:%[^:]:%[^:]: %ld %s %s %s',
        &host, &app, &sub_app, &metric_name, &timestamp,
        &valuestr, &storage_type, &source_type);
    if(!strcmp(storage_type, "double")) {
        double_value = atof(valuestr);
    } else {
        int_value = atol(valuestr);
    }


Example in Python
-----------------

The following is a brief example for python parser. Note: no regular
expressions needed. ::

    head = source_message.split('\n', 1)[0]
    metric, timestamp, valuestr, stor_type, src_type = head.split(None)[:5]
    prefix, host, app, sub_app, metric_name, _empty = metric.split(':')
    if stor_type == 'double':
        value = float(valuestr)
    else:
        value = int(valuestr)


Copyright
=========

This document has been placed in the public domain.
