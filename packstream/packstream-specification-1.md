---
layout: default
---
# PackStream Specification

* [**Version 1**](#version1)

# Version 1

PackStream is a general purpose data serialisation format, originally inspired by (but incompatible with) [MessagePack](http://msgpack.org).

The format provides a type system that is fully compatible with the [types supported by Cypher](https://neo4j.com/docs/cypher-manual/current/syntax/values/).

PackStream offers a number of core data types, many supported by multiple binary representations, as well as a flexible extension mechanism.

The core data types are as follows:

| Data Type                   | Description                                           |
|-----------------------------|-------------------------------------------------------|
| [`Null`](#null)             | missing or empty value                                |
| [`Boolean`](#boolean)       | **true** or **false**                                 |
| [`Integer`](#integer)       | signed 64-bit integer                                 |
| [`Float`](#float)           | 64-bit floating point number                          |
| [`Bytes`](#bytes)           | byte array                                            |
| [`String`](#string)         | unicode text, **UTF-8**                               |
| [`List`](#list)             | ordered collection of values                          |
| [`Dictionary`](#dictionary) | collection of key-value entries (no order guaranteed) |
| [`Structure`](#structure)   | composite value with a type signature                 |

**NOTE:** Neither unsigned integers nor 32-bit floating point numbers are included.
This is a deliberate design decision to allow broader compatibility across client languages.


The PackStream specified structures:

| Structure Name                                               | Description                                                                                                                                                           |
|--------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [`Node`](#node---structure)                                  | snapshot of a node within a graph database                                                                                                                            |
| [`Relationship`](#relationship---structure)                  | snapshot of a relationship within a graph database                                                                                                                    |
| [`UnboundRelationship`](#unboundrelationship---structure)    | relationship detail without start or end node information                                                                                                             |
| [`Path`](#path---structure)                                  | alternating sequence of nodes and relationships                                                                                                                       |
| [`Date`](#date---structure)                                  | a date without a time-zone in the ISO-8601 calendar system, such as "2007-12-03"                                                                                      |
| [`Time`](#time---structure)                                  | a time with an offset from UTC/Greenwich in the ISO-8601 calendar system, such as "10:15:30+01:00"                                                                    |
| [`LocalTime`](#localtime---structure)                        | a time without a time-zone in the ISO-8601 calendar system, such as "10:15:30"                                                                                        |
| [`DateTime`](#datetime---structure)                          | a date-time with a time-zone in the ISO-8601 calendar system, such as "2007-12-03T10:15:30+01:00 Europe/Paris", the time-zone is specified in minutes offset from UTC |
| [`DateTimeZoneId`](#datetimezoneid---structure)              | a date-time with a time-zone in the ISO-8601 calendar system, such as "2007-12-03T10:15:30+01:00 Europe/Paris", the time-zone is specified with a **zone id**         |
| [`LocalDateTime`](#localdatetime---structure)                | a date-time without a time-zone in the ISO-8601 calendar system, such as "2007-12-03T10:15:30"                                                                        |
| [`Duration`](#duration---structure)                          | a temporal amount                                                                                                                                                     |
| [`Point2D`](#point2d---structure)                            | represents a single location in 2-dimensional space                                                                                                                   |
| [`Point3D`](#point3d---structure)                            | represents a single location in 3-dimensional space                                                                                                                   |


## General Representation

Every serialised PackStream value begins with a **marker byte**.

The marker contains data type information as well as direct or indirect size information for types that require it.
How that size information is encoded varies by marker type.

Some values, such as Boolean true, can be encoded within a single marker byte.
Many small integers (specifically between -16 and +127 inclusive) are also encoded within a single byte.

A number of marker bytes are reserved for future expansion of the format itself.
These bytes should not be used, and encountering them in an incoming stream should treated as an error.


### Sized Values

Some representations are of a variable length and, as such, have their size explicitly encoded in the representation.
Such values generally begin with a single marker byte, followed by a size, followed by the data content itself.
In this context, the marker denotes both type and scale and therefore determines the number of bytes used to represent the size of the data.
The size itself is encoded as either an 8-bit, 16-bit or 32-bit unsigned integer.
Sizes longer than this are not supported.

The diagram below illustrates the general layout for a sized value, here with a 16-bit size:


[//]: Marker   Size           Content

[//]:  `<>`  `<--->`  `<--------------------->`

[//]:  `XX`  `XX XX`  `XX XX XX XX .. .. .. XX`


![alt text](../art/packstream_sized_value.png "sized value")


### Endianness

PackStream exclusively uses [big-endian](https://en.wikipedia.org/wiki/Endianness#Big-endian) representations.
This means that the most significant part of the value is written to the network or memory space first and the least significant part is written last.


## Null

**Marker:** `C0`

Null is always encoded using the single marker byte `C0`.


## Boolean

**Marker, false:** `C2`

**Marker, true:** `C3`


Boolean values are encoded within a single marker byte, using `C3` to denote true and `C2` to denote false.


## Integer

**Markers, TINY_INT:**

| Marker  | Decimal Number |
|---------|---------------:|
| `F0`    | -16            |
| `F1`    | -15            |
| `F2`    | -14            |
| `F3`    | -13            |
| `F4`    | -12            |
| `F5`    | -11            |
| `F6`    | -10            |
| `F7`    | -9             |
| `F8`    | -8             |
| `F9`    | -7             |
| `FA`    | -6             |
| `FB`    | -5             |
| `FC`    | -4             |
| `FD`    | -3             |
| `FE`    | -2             |
| `FF`    | -1             |
| `00`    | 0              |
| `01`    | 1              |
| `02`    | 2              |
| ...     | ...            |
| ...     | ...            |
| ...     | ...            |
| `7E`    | 126            |
| `7F`    | 127            |


**Marker, INT_8:** `C8`

**Marker, INT_16:** `C9`

**Marker, INT_32:** `CA`

**Marker, INT_64:** `CB`


Integer values occupy either 1, 2, 3, 5 or 9 bytes depending on magnitude.

The available representations are:

| Representation   | Size (bytes) | Description
|------------------|--------------|----------------------------------------------
| `TINY_INT`       | 1            | marker byte only
| `INT_8`          | 2            | marker byte `C8` followed by signed 8-bit integer
| `INT_16`         | 3            | marker byte `C9` followed by signed 16-bit integer
| `INT_32`         | 5            | marker byte `CA` followed by signed 32-bit integer
| `INT_64`         | 9            | marker byte `CB` followed by signed 64-bit integer


The available encodings are illustrated below and each shows a valid representation for the **decimal value 42**:


| Representation   | Size (bytes) | Decimal Value 42
|------------------|--------------|----------------------------------------------
| `TINY_INT`       | 1            | `2A`
| `INT_8`          | 2            | `C8 2A`
| `INT_16`         | 3            | `C9 00 2A`
| `INT_32`         | 5            | `CA 00 00 00 2A`
| `INT_64`         | 9            | `CB 00 00 00 00 00 00 00 2A`


Some marker bytes can be used to carry the value of a small integer as well as its type.

These markers can be identified by a zero high-order bit (for positive values) or by a high-order nibble containing only ones (for negative values).

Specifically, values between `00` and `7F` inclusive can be directly translated to and from positive integers with the same value.

Similarly, values between `F0` and `FF` inclusive can do the same for negative numbers between -16 and -1.

Note that while it is possible to encode small numbers in wider formats, it is generally recommended to use the most compact representation possible.

The following table shows the **optimal representation** for every possible integer in the signed 64-bit range:

| Range Minimum              |  Range Maximum             | Optimal Representation |
|----------------------------|----------------------------|------------------------|
| -9 223 372 036 854 775 808 |             -2 147 483 649 | `INT_64`               |
|             -2 147 483 648 |                    -32 769 | `INT_32`               |
|                    -32 768 |                       -129 | `INT_16`               |
|                       -128 |                        -17 | `INT_8`                |
|                        -16 |                       +127 | `TINY_INT`             |
|                       +128 |                    +32 767 | `INT_16`               |
|                    +32 768 |             +2 147 483 647 | `INT_32`               |
|             +2 147 483 648 | +9 223 372 036 854 775 807 | `INT_64`               |



Example 1:

Value: `-9223372036854775808` (The minimum)

```
CB 80 00 00 00 00 00 00 00
```

Example 2:

Value: `9223372036854775807` (The maximum)

```
CB 7F FF FF FF FF FF FF FF
```


## Float

**Marker:** `C1`

Floats are double-precision floating-point values, generally used for representing fractions and decimals.

Floats are encoded as a single `C1` marker byte followed by 8 bytes which are formatted according to the IEEE 754 floating-point "double format" bit layout in big-endian order.

+ Bit 63 (the bit that is selected by the mask `0x8000000000000000`) represents the sign of the number.
+ Bits 62-52 (the bits that are selected by the mask `0x7ff0000000000000`) represent the exponent.
+ Bits 51-0 (the bits that are selected by the mask `0x000fffffffffffff`) represent the significand (sometimes called the mantissa) of the number.

The value **1.23 in decimal** can be represented as:

```
C1 3F F3 AE 14 7A E1 47 AE
```

## Bytes

Bytes are arrays of byte values.

These are used to transmit raw binary data and the size represents the number of bytes contained.

Unlike other values, there is no separate encoding for byte arrays containing fewer than 16 bytes.

| Marker | Size                               | Maximum Size        |
|--------|------------------------------------|---------------------|
| `CC`   | 8-bit big-endian unsigned integer  | 255 bytes           |
| `CD`   | 16-bit big-endian unsigned integer | 65 535 bytes        |
| `CE`   | 32-bit big-endian signed integer   | 2 147 483 648 bytes |

One of the markers `CC`, `CD` or `CE` should be used, depending on scale.

This marker is followed by the size and bytes themselves.

Example 1:

Empty byte array `b[]`

```
CC 00
```

Example 2:

Byte array containing three values 1, 2 and 3; `b[1, 2, 3]`

```
CC 03 01 02 03
```

## String

**Markers:**

For shorter strings,

| Marker  | Size (bytes)   |
|---------|----------------|
| `80`    | 0              |
| `81`    | 1              |
| `82`    | 2              |
| `83`    | 3              |
| `84`    | 4              |
| `85`    | 5              |
| `86`    | 6              |
| `87`    | 7              |
| `88`    | 8              |
| `89`    | 9              |
| `8A`    | 10             |
| `8B`    | 11             |
| `8C`    | 12             |
| `8D`    | 13             |
| `8E`    | 14             |
| `8F`    | 15             |


For longer strings,

| Marker  | Size                                | Maximum number of bytes |
|---------|-------------------------------------|-------------------------|
| `D0`    | 8-bit big-endian unsigned integer   | 255 bytes               |
| `D1`    | 16-bit big-endian unsigned integer  | 65 535 bytes            |
| `D2`    | 32-bit big-endian signed integer    | 2 147 483 648 bytes     |


Text data is represented as **UTF-8** encoded bytes.

**Note:** The sizes used in string representations are the byte counts of the UTF-8 encoded data, not the character count of the original text.

For encoded text containing fewer than 16 bytes, including empty strings, the marker byte should contain the high-order nibble '8' (binary 1000) followed by a low-order nibble containing the size. The encoded data then immediately follows the marker.

For encoded text containing 16 bytes or more, the marker `D0`, `D1` or `D2` should be used, depending on scale.
This marker is followed by the size and the UTF-8 encoded data.


Examples follow below:

| Value                                       | Encoding                                                                               |
|---------------------------------------------|----------------------------------------------------------------------------------------|
| `String("")`                                | `80`                                                                                   |
| `String("A")`                               | `81 41`                                                                                |
| `String("ABCDEFGHIJKLMNOPQRSTUVWXYZ")`      | `D0 1A 41 42 43 44 45 46 47 48 49 4A 4B 4C 4D 4E 4F 50 51 52 53 54 55 56 57 58 59 5A`  |
| `String("Größenmaßstäbe")`                  | `D0 12 47 72 C3 B6 C3 9F 65 6E 6D 61 C3 9F 73 74 C3 A4 62 65`                          |


## List

Lists are heterogeneous sequences of values and therefore permit a mixture of types within the same list.
The size of a list denotes the number of items within that list, rather than the total packed byte size.

The markers used to denote a list are described in the table below:


| Marker | Size (items)                            | Maximum size          |
|--------|-----------------------------------------|-----------------------|
| `90`   | the low-order nibble of marker          | 0 items               |
| `91`   | the low-order nibble of marker          | 1 items               |
| `92`   | the low-order nibble of marker          | 2 items               |
| `93`   | the low-order nibble of marker          | 3 items               |
| `94`   | the low-order nibble of marker          | 4 items               |
| `95`   | the low-order nibble of marker          | 5 items               |
| `96`   | the low-order nibble of marker          | 6 items               |
| `97`   | the low-order nibble of marker          | 7 items               |
| `98`   | the low-order nibble of marker          | 8 items               |
| `99`   | the low-order nibble of marker          | 9 items               |
| `9A`   | the low-order nibble of marker          | 10 items              |
| `9B`   | the low-order nibble of marker          | 11 items              |
| `9C`   | the low-order nibble of marker          | 12 items              |
| `9D`   | the low-order nibble of marker          | 13 items              |
| `9E`   | the low-order nibble of marker          | 14 items              |
| `9F`   | the low-order nibble of marker          | 15 items              |
| `D4`   | 8-bit big-endian unsigned integer       | 255 items             |
| `D5`   | 16-bit big-endian unsigned integer      | 65 535 items          |
| `D6`   | 32-bit big-endian signed integer        | 2 147 483 648 items   |


For lists containing fewer than 16 items, including empty lists, the marker byte should contain the high-order nibble '9' (binary 1001) followed by a low-order nibble containing the size.
The items within the list are then serialised in order immediately after the marker.

For lists containing 16 items or more, the marker `D4`, `D5` or `D6` should be used, depending on scale.
This marker is followed by the size and list items, serialized in order.


Example 1:

``` 
[]
```

```
90
```


Example 2:

```
[Integer(1), Integer(2), Integer(3)]
```

```
93 01 02 03
```


Example 3:

```
[
  Integer(1),
  Float(2.0),
  String("three")
]
```

```
93
01
C1 40 00 00 00 00 00 00 00
85 74 68 72 65 65
```

Example 4:

```
[
    Integer(1),
    Integer(2),
    ...
    Integer(40)
]
```

```
D4 28
01 02 03 04 05 06 07 08 09 0A
0B 0C 0D 0E 0F 10 11 12 13 14
15 16 17 18 19 1A 1B 1C 1D 1E
1F 20 21 22 23 24 25 26 27 28
```


## Dictionary

A Dictionary is a list containing key-value entries.

* Keys must be a String.

* Can contain multiple instances of the same key.

* Permit a mixture of types.

The size of a Dictionary denotes the number of key-value entries within that Dictionary, not the total packed byte size.

The markers used to denote a Dictionary are described in the table below:


| Marker | Size (key-value entries)                     | Maximum size             |
|--------|----------------------------------------------|--------------------------|
| `A0`   | contained within low-order nibble of marker  | 0                        |
| `A1`   | contained within low-order nibble of marker  | 1                        |
| `A2`   | contained within low-order nibble of marker  | 2                        |
| `A3`   | contained within low-order nibble of marker  | 3                        |
| `A4`   | contained within low-order nibble of marker  | 4                        |
| `A5`   | contained within low-order nibble of marker  | 5                        |
| `A6`   | contained within low-order nibble of marker  | 6                        |
| `A7`   | contained within low-order nibble of marker  | 7                        |
| `A8`   | contained within low-order nibble of marker  | 8                        |
| `A9`   | contained within low-order nibble of marker  | 9                        |
| `AA`   | contained within low-order nibble of marker  | 10                       |
| `AB`   | contained within low-order nibble of marker  | 11                       |
| `AC`   | contained within low-order nibble of marker  | 12                       |
| `AD`   | contained within low-order nibble of marker  | 13                       |
| `AE`   | contained within low-order nibble of marker  | 14                       |
| `AF`   | contained within low-order nibble of marker  | 15                       |
| `D8`   | 8-bit big-endian unsigned integer            | 255 entries              |
| `D9`   | 16-bit big-endian unsigned integer           | 65 535 entries           |
| `DA`   | 32-bit big-endian signed integer             | 2 147 483 648 entries    |


For a Dictionary containing fewer than 16 key-value entries, including an empty Dictionary,
the marker byte should contain the high-order nibble `A` (binary `1010`) followed by a low-order nibble containing the size.

The entries within the Dictionary are then serialised in `[key, value, key, value]` order immediately after the marker.

**Keys are always String values.**

For a Dictionary containing 16 key-value entries or more, the marker `D8`, `D9` or `DA` should be used, depending on scale.
This marker is followed by the size and the key-value entries.

Example 1:

```
{}
```

```
A0
```

Example 2:

```
{"one": "eins"}
```

```
A1 83 6F 6E 65 84 65 69 6E 73
```

Example 3:

```
{"A": 1, "B": 2 ... "Z": 26}
```

```
D8 1A
81 41 01 81 42 02 81 43 03 81 44 04
81 45 05 81 46 06 81 47 07 81 48 08
81 49 09 81 4A 0A 81 4B 0B 81 4C 0C
81 4D 0D 81 4E 0E 81 4F 0F 81 50 10
81 51 11 81 52 12 81 53 13 81 54 14
81 55 15 81 56 16 81 57 17 81 58 18
81 59 19 81 5A 1A
```

When unpacked if there are multiple instances of the same key, the last seen value for that key should be used.

Example:

```
[("key_1", 1), ("key_2", 2), ("key_1", 3)] -> {"key_1": 3, "key_2": 2}
```


## Structure


A structure is a composite value, comprised of fields and a unique type code.

Structure encodings consist, beyond the marker, of a single byte, the **tag byte**, followed by a sequence of up to 15 fields, each an individual value.

The size of a structure is measured as the number of fields and not the total byte size.

This count does not include the tag.

The markers used to denote a structure are described in the table below:

| Marker | Size (fields)                               | Maximum size |
|--------|---------------------------------------------|--------------|
| `B0`   | contained within low-order nibble of marker | 0 fields     |
| `B1`   | contained within low-order nibble of marker | 1 field      |
| `B2`   | contained within low-order nibble of marker | 2 fields     |
| `B3`   | contained within low-order nibble of marker | 3 fields     |
| `B4`   | contained within low-order nibble of marker | 4 fields     |
| `B5`   | contained within low-order nibble of marker | 5 fields     |
| `B6`   | contained within low-order nibble of marker | 6 fields     |
| `B7`   | contained within low-order nibble of marker | 7 fields     |
| `B8`   | contained within low-order nibble of marker | 8 fields     |
| `B9`   | contained within low-order nibble of marker | 9 fields     |
| `BA`   | contained within low-order nibble of marker | 10 fields    |
| `BB`   | contained within low-order nibble of marker | 11 fields    |
| `BC`   | contained within low-order nibble of marker | 12 fields    |
| `BD`   | contained within low-order nibble of marker | 13 fields    |
| `BE`   | contained within low-order nibble of marker | 14 fields    |
| `BF`   | contained within low-order nibble of marker | 15 fields    |


For structures containing fewer than 16 fields, the marker byte should contain the high-order nibble 'B' (binary 1011) followed by a low-order nibble containing the size.

The marker is immediately followed by the **tag byte** and the field values in that order.

The **tag byte** is used to identify the type or class of the structure.

The **tag byte** may hold any value between 0 and +127.


| Structure Name                                            | Code | tag byte | Description                                                                                                                                                           |
|-----------------------------------------------------------|------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| [`Node`](#node---structure)                               | 'N'  | `4E`     | snapshot of a node within a graph database                                                                                                                            |
| [`Relationship`](#relationship---structure)               | 'R'  | `52`     | snapshot of a relationship within a graph database                                                                                                                    |
| [`UnboundRelationship`](#unboundrelationship---structure) | 'r'  | `72`     | relationship detail without start or end node information                                                                                                             |
| [`Path`](#path---structure)                               | 'P'  | `50`     | alternating sequence of nodes and relationships                                                                                                                       |
| [`Date`](#date---structure)                               | 'D'  | `44`     | a date without a time-zone in the ISO-8601 calendar system, such as "2007-12-03"                                                                                      |
| [`Time`](#time---structure)                               | 'T'  | `54`     | a time with an offset from UTC/Greenwich in the ISO-8601 calendar system, such as "10:15:30+01:00"                                                                    |
| [`LocalTime`](#localtime---structure)                     | 't'  | `74`     | a time without a time-zone in the ISO-8601 calendar system, such as "10:15:30"                                                                                        |
| [`DateTime`](#datetime---structure)                       | 'F'  | `46`     | a date-time with a time-zone in the ISO-8601 calendar system, such as "2007-12-03T10:15:30+01:00 Europe/Paris", the time-zone is specified in minutes offset from UTC |
| [`DateTimeZoneId`](#datetimezoneid---structure)           | 'f'  | `66`     | a date-time with a time-zone in the ISO-8601 calendar system, such as "2007-12-03T10:15:30+01:00 Europe/Paris", the time-zone is specified with a **zone id**         |
| [`LocalDateTime`](#localdatetime---structure)             | 'd'  | `64`     | a date-time without a time-zone in the ISO-8601 calendar system, such as "2007-12-03T10:15:30"                                                                        |
| [`Duration`](#duration---structure)                       | 'E'  | `45`     | a temporal amount                                                                                                                                                     |
| [`Point2D`](#point3d---structure)                         | 'X'  | `58`     | represents a single location in 2-dimensional space                                                                                                                   |
| [`Point3D`](#point3d---structure)                         | 'Y'  | `59`     | represents a single location in 3-dimensional space                                                                                                                   |


### Node - Structure

**tag byte:** `4E`

**Number of fields:** 3

```
Node::Structure(
    id::Integer,
    labels::List<String>,
    properties::Dictionary,
)
```

Example:

```
Node(
  id = 3,
  labels = ["Example", "Node"],
  properties = {"name": "example"},
)
```

```
B3 4E
...
```

### Relationship - Structure

**tag byte:** `52`

**Number of fields:** 5

```
Relationship::Structure(
    id::Integer,
    startNodeId::Integer,
    endNodeId::Integer,
    type::String,
    properties::Dictionary,
)
```

Example:

```
Relationship(
    id = 11,
    startNodeId = 2,
    endNodeId = 3,
    type = "KNOWS",
    properties = {"name": "example"},
)
```

```
B5 52
...
```

### UnboundRelationship - Structure

**tag byte:** `72`

**Number of fields:** 3

A relationship without information of start and end node id. It is used internally for Path serialization. 

```
UnboundRelationship::Structure(
    id::Integer,
    type::String,
    properties::Dictionary,
)
```

Example:

```
UnboundRelationship(
    id = 17,
    type = "KNOWS",
    properties = {"name": "example"},
)
```

```
B3 72
...
```

### Path - Structure

**tag byte:** `50`

**Number of fields:** 3

```
Path::Structure(
    nodes::List<Node>,
    rels::List<UnboundRelationship>,
    ids::List<Integer>,
)
```

- The `rels` field is a list of unbound relationships.
- The `ids` is a list of relationship id and node id to represent the path.

### Date - Structure

**tag byte:** `44`

**Number of fields:** 1

An instant capturing the date, but not the time, nor the time zone.

```
Date::Structure(
    days::Integer,
)
```

- The `days` are days since the Unix epoch.


### Time - Structure

An instant capturing the time of day, and the timezone, but not the date.

**tag byte:** `54`

**Number of fields:** 2

```
Time::Structure(
    nanoseconds::Integer,
    tz_offset_seconds::Integer,
)
```

- The `nanoseconds` are nanoseconds since midnight. This time is not [UTC](https://en.wikipedia.org/wiki/Coordinated_Universal_Time).
- The `tz_offset_seconds` are an offset in seconds from UTC.


To convert to UTC use:

```
utc_nanoseconds = nanoseconds - (tz_offset_seconds * 1000000000)
```


### LocalTime - Structure

**tag byte:** `74`

**Number of fields:** 1

An instant capturing the time of day, but not the date, nor the time zone.

```
LocalTime::Structure(
    nanoseconds::Integer,
)
```

- The `nanoseconds` are nanoseconds since midnight.

### DateTime - Structure

**tag byte:** `46`

**Number of fields:** 3

An instant capturing the date, the time, and the time zone.

The time zone information is specified with a zone offset.

```
DateTime::Structure(
    seconds::Integer,
    nanoseconds::Integer,
    tz_offset_seconds::Integer,
)
```

- The `seconds` are seconds since the adjusted [Unix epoch](https://en.wikipedia.org/wiki/Epoch_(computing)). This is not [UTC](https://en.wikipedia.org/wiki/Coordinated_Universal_Time).
- The `tz_offset_seconds` specifies the offset in seconds from UTC.


To convert to UTC:

```
utc_nanoseconds = (seconds * 1000000000) + nanoseconds - (tx_offset_seconds * 1000000000)
```

### DateTimeZoneId - Structure

**tag byte:** `66`

**Number of fields:** 3

An instant capturing the date, the time, and the time zone.

The time zone information is specified with a **zone identifier**.

```
DateTimeZoneId::Structure(
    seconds::Integer,
    nanoseconds::Integer,
    tz_id::String,
)
```

- The `seconds` are seconds since the adjusted [Unix epoch](https://en.wikipedia.org/wiki/Epoch_(computing)). This is not [UTC](https://en.wikipedia.org/wiki/Coordinated_Universal_Time).
- The `tz_id` is an identifier for a specific time zone, such as `"Europe/Paris"`.

To convert to UTC:

```
utc_nanoseconds = (seconds * 1000000000) + nanoseconds - get_offset_in_nanoseconds(tz_id)
```

### LocalDateTime - Structure

**tag byte:** `64`

**Number of fields:** 2

An instant capturing the date and the time, but not the time zone.

```
LocalDateTime::Structure(
    seconds::Integer,
    nanoseconds::Integer,
)
```

- The `seconds` are seconds since the Unix epoch.

### Duration - Structure

**tag byte:** `45`

**Number of fields:** 4

A temporal amount. This captures the difference in time between two instants.
It only captures the amount of time between two instants, it does not capture a start time and end time.
A unit capturing the start time and end time would be a Time Interval and is out of scope for this proposal.

A Duration can be negative.

```
Duration::Structure(
    months::Integer,
    days::Integer,
    seconds::Integer,
    nanoseconds::Integer,
)
```

### Point2D - Structure

**tag byte:** `58`

**Number of fields:** 3

Represents a single location in 2-dimensional space.

```
Point2D::Structure(
    srid::Integer,
    x::Float,
    y::Float,
)
```

- The `srid` is a [Spatial Reference System Identifier](https://en.wikipedia.org/wiki/Spatial_reference_system#Identifier).


### Point3D - Structure

**tag byte:** `59`

**Number of fields:** 4

Represents a single location in 3-dimensional space.

```
Point3D::Structure(
    srid::Integer,
    x::Float,
    y::Float,
    z::Float,
)
```

- The `srid` is a [Spatial Reference System Identifier](https://en.wikipedia.org/wiki/Spatial_reference_system#Identifier).

