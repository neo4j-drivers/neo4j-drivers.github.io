# PackStream v1 Specification

PackStream is a general purpose data serialisation format, originally inspired by (but incompatible with) [MessagePack](http://msgpack.org).
The format provides a type system that is fully compatible with the [universal type system](type-system.md) used by [Neo4j](http://neo4j.com).

PackStream offers a number of core data types, many supported by multiple binary representations, as well as a flexible extension mechanism.
The core data types are as follows:

| Data Type | Description
|-----------|----------------------
| Null      | Missing or empty value
| Boolean   | True or false
| Integer   | Signed 64-bit integer
| Float     | 64-bit floating point number
| Bytes     | Byte array
| String    | Unicode text
| List      | Ordered collection of values
| Map       | Keyed collection of values
| Structure | Composite value with a type signature

NOTE: Neither unsigned integers nor 32-bit floating point numbers are included.
This is a deliberate design decision to allow broader compatibility across client languages.


## General Representation

Every serialised PackStream value begins with a _marker byte_.
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

```
Marker Size          Content
  <>   <--->  <--------------------->
  XX   XX XX  XX XX XX XX .. .. .. XX
```

### Endianness

Packstream exclusively uses https://en.wikipedia.org/wiki/Endianness#Big-endian[big-endian] representations.
This means that the most significant part of the value is written to the network or memory space first and the least significant part is written last.


## Null

Null is always encoded using the single marker byte `C0`.


## Boolean

Boolean values are encoded within a single marker byte, using `C3` to denote true and `C2`
to denote false.


## Integer

Integer values occupy either 1, 2, 3, 5 or 9 bytes depending on magnitude.
The available representations are:

| Representation | Size (bytes) | Description
|----------------|--------------|---------------------------------------------
| TINY_INT       | 1            | Marker byte only
| INT_8          | 2            | Marker byte followed by `int8_t` value byte
| INT_16         | 3            | Marker byte followed by `int16_t` value byte
| INT_32         | 5            | Marker byte followed by `int32_t` value byte
| INT_64         | 9            | Marker byte followed by `int64_t` value byte

Some marker bytes can be used to carry the value of a small integer as well as its type.
These markers can be identified by a zero high-order bit (for positive values) or by a high-order nibble containing only ones (for negative values).
Specifically, values between 0x00 and 0x7F inclusive can be directly translated to and from positive integers with the same value.
Similarly, values between 0xF0 and 0xFF inclusive can do the same for negative numbers between -16 and -1.

Note that while it is possible to encode small numbers in wider formats, it is generally recommended to use the most compact representation possible.
The following table shows the optimal representation for every possible integer in the signed 64-bit range:

| Range Minimum              |  Range Maximum             | Optimal Representation
|----------------------------|----------------------------|------------------------
| -9 223 372 036 854 775 808 |             -2 147 483 649 | INT_64
|             -2 147 483 648 |                    -32 769 | INT_32
|                    -32 768 |                       -129 | INT_16
|                       -128 |                        -17 | INT_8
|                        -16 |                       +127 | TINY_INT
|                       +128 |                    +32 767 | INT_16
|                    +32 768 |             +2 147 483 647 | INT_32
|             +2 147 483 648 | +9 223 372 036 854 775 807 | INT_64


## Float


## Bytes


## String


## List


## Map


## Structure
