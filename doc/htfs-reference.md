# HTFS: Human-editable Transit Feed Specification.

This document is a work-in progress proposal for a specification for
exchanging timetable information in an environment where this information
is created manually rather than derived from automatically processing data
stored in databases.

## Introduction

The data model used for HTFS is compatible with the [General Transit Feed
Specification](https://gtfs.org/), specifically with the [GTFS
Schdedule](https://gtfs.org/schedule/) specification for static data. In
essence, we have taken the data model from this specification and
converted the data format from CSV to a subset of the YAML markup language
to make it more convenient to edit data.

As this is an early proposal made for a very specific use-case, the
following specification only covers those parts of the GTFS Schedule
specification that we feel we need.


## Dataset

A dataset consists of any number of YAML files. Files can be made
available through a single ZIP file, through a repository, or any other
means that allows distributing a set of files.

Each YAML file consists of a number of documents. The dataset is the sum
of all documents from all the files.


## Documents

Documents are portions of a YAML file started with three dashes on their
own line and ended with either the end of a file or the start of a new
document. Each document is a YAML mapping with string keys and simple or
complex values as described below. We call these key/value pairs _fields_.

The type of a document is provided via the `type` key. For each type, a
description is given below.

## Field Types

The field types are the same as the field types in the GTFS Schedule
definition with the following exceptions:

* **Localized Text** – Optionally localized text. This can either be a
  simple YAML string with text to be used for all languages or a YAML
  mapping where each key is a _language code_ and the value is the text
  for that particular language. The key `default` allows specifying a
  fallback value for all languages not explicitly specified.

  This type replaces the `translations.txt` file from the GTFS Schedule
  specification.

* For **enum** types, we generally used strings rather than number to make
  it easier to remember the options. The conversion between HTFS strings
  and GTFS numbers is given below.


## Document Definitions

### `type: agency` – Transit Agencies

Transit agencies are operators of transit services. Each agency document
describes one operator. The fields are identical to the fields in the
`agency.txt` file of the GTFS Schedule specification.


### `type: stop` – Stop Definitions

A stop document describes a stop to be used in schedule information. The
fields in this document are identical to those of the `stops.txt` file of
the GTFS Schedule specification with the following exceptions:

The `parent_station` field which is omitted. Instead, dependent stops are
collected in a single document. The optional field `includes` is sequence of
mappings with the same fields as the stop document itself (including a
nested `includes` field) which contains all the stops that would have the
stop as their parent.

The type of the `stop_code` field is changed to _localized text_.

The `location_type` field uses strings instead of numbers to identify the
variants of the enum. These are `stop` for GTFS value 0, `station` for GTFS
value 1, `entrance` or `exit` for GTFS value 2, `node` for GTFS value 3, and `boarding` for GTFS value 4.

The `wheelchair_boarding` field uses strings instead numbers to identify
the variants of the enum. These are `none` for a station, entrance, or
platform that is not accessible to wheelchair users (GTFS value 2),
`partial` for GTFS value 1 for parentless stops, and `available` for GTFS
value 1 for platforms or entrances that are accessible.


### `type: route` - Route Definitions

A route is a collection of trips. The fields in this document are
identical to those of the `routes.txt` file of the GTFS Schedule
specification with the following rather significant exceptions.

The type of the `route_short_name`, `route_long_name`, and `route_desc`
fields are changed to _localized text._

The `route_type` enum uses the following strings: `tram` (0), `metro` (1),
`rail` (2), `bus` (3), `ferry` (4), `cable_tram` (5), `aerial` (6),
`funicular` (7), `trolleybus` (11), `monorail` (12).

The `continuous_pickup` and `continuous_drop_off` enums use the following
strings: `full` (0), `none` (1), `phone` (2), `driver` (3).

In addition, the field `trips` is a YAML sequence containing mappings with
fields for the `trips.txt` file of the GTFS Schedule specification with
the following exceptions:

The `route_id` field is omitted.

The `direction_id` enum uses the strings `up` (0) and `down` (1).

The `wheelchair_accessible` and `bikes_allowed` enums use the strings
`unknown` (0), `yes` (1), `none` (2).

In addition the field `stops` is a YAML sequence containing mappings with
the fields for the `stop_times.txt` file fo the GTFS Schedule
specification with the following exception:

The `trip_id` field is omitted.

The `stop_sequence` field is omitted as it can be generated from the
position of the stop in the YAML sequence.

The `pickup_type`, `drop_off_type`,  `continuous_pickup`, and
`continuous_drop_off` enums use the strings `full` (0), `none` (1),
`phone` (2), and `driver` (3).

The `timepoint` field is replaced by a boolean field `approximate`.


### `type: calendar` – Service Date Definitions

A service date definition describes the dates at which a service operates.
This relates to the `calendar.txt` and `calendar_dates.txt` files of the
GTFS Schedule specification but uses a slightly different strategy.

The service date definition has the following fields:

* `service_id` is a unique ID of for the definition and is required.

* `inherits` is an optional field containing a reference or a YAML sequence
  references to a service definitions.  If provided, these definitions will
  be used as a basis for the new service definition. See below for details.

* `start_date` and `end_date` are date values defining the first and last
  day the service will run. Unlike in GTFS, these fields are optional.
  However, if they are not given in any of the inherited services or the
  service itself, the service cannot be used for trips.

* `also_weekdays` is a sequence of strings describing the weekdays this
  service will also run at. The strings are `mo`, `tu`, `we`, `th`, `fr`,
  `sa`, `su` for the various weekdays. Each string must only be present
  once. Alternatively, the string `all` can be used instead of the
  sequence as a shortcut for a sequence including all days.

* `not_weekdays` is a sequence of strings describing the weekdays this
  service will not run at. The same strings as for `also_weekdays` is
  used.

* `also_dates` is a sequence of dates this service will also run at.

* `not_dates` is a sequence of dates this service will not run at.

The actual service definition will be constructed by starting with an
empty service definition, i.e., a service that never operates, and
applying each inherited service in order of their appearance in the
sequence and finally the definition itself. For each step, the following
operations are performed in this order:

* the `start_date` and `end_date` are overwritten if present;

* all weekdays mentioned in the `also_weekdays` field are added to the set of
  weekdays this service runs on;

* all weekdays mentioned in the `not_weekdays` field are removed from the
  set of weekdays the service runs on;

* each date mentioned in the `also_dates` field that is part of the set of
  excluded dates is removed from this set;

* each remaining date in the `also_dates` field is added to the set of
  additional dates;

* each date mentioned in the `not_dates` field that is part of the set of
  additional dates is removed from this set;

* each remaining date in the `not_dates` field is added to the set of
  excluded dates.

Note that because of this order weekdays and dates that appear in both the
also and not fields will be excluded.


