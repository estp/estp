===========================
Collectd Extension for ESTP
===========================

:Author: Paul Colomiets <paul@colomiets.name>
:State: draft
:Version: 0.1


The goal of this extension is to provide lossless export of collectd data to
ESTP protocol, being able to provide standard protocol for applications that
doesn't need collectd-specific parts.

Example of extension data::

    ESTP:teapot:load::shortterm: 2012-06-11T21:36:57 10 0.37
     :collectd: type=load items=shortterm,midterm,longterm

The above specifies that time collectd should use is ``load`` and that three
values should be combined to form single collectd value


Extension Structure
===================

Extension name is "collectd". According to ESTP specification, the extension
line must start with `` :collectd:`` (note the space at the start). Following
is a list of key-value pairs delimited equals (``=``) sign. The keys are:

* ``type`` -- the collectd type of the value, must exists in types.db if
  different from "gauge", "derive", "absolute"

* ``items`` -- the comma-separaged suffixes of the metrics, that should be
  combined to form the collectd type, if type has more than one value

Note if the type is one of the types that ESTP supports natively, the whole
extension line should be omitted. The ``items`` with single item is not allowed.
The ``items`` field without ``type`` is meaningless, so is prohibited too.

The "suffix" that contained in ``items`` is a substring of metric name (part
between the last pair of the colons of metric full name) starting from last dot
to the end or the whole metric name if no dot exists in metric name.

Only one extension line starting with ``:collectd:`` is allowed. (Only first
such line is parsed by collectd plugin)


Copyright
=========

This document has been placed in the public domain.

For the legal description of the statement above see:

http://creativecommons.org/publicdomain/zero/1.0/


