# PackStream Specification

* [Version 1](#version1)

# Version 1

PackStream is a general purpose data serialisation format, originally inspired by (but incompatible with) [MessagePack](http://msgpack.org).

The format provides a type system that is fully compatible with the [types supported by Cypher](https://neo4j.com/docs/cypher-manual/current/syntax/values/).

PackStream offers a number of core data types, many supported by multiple binary representations, as well as a flexible extension mechanism.

The core data types are as follows:

| Data Type   | Description                                   |
|-------------|-----------------------------------------------|
| `Null`      | missing or empty value                        |
| `Boolean`   | **true** or **false**                         |
| `Integer`   | signed 64-bit integer                         |
| `Float`     | 64-bit floating point number                  |
| `Bytes`     | byte array                                    |
| `String`    | unicode text, **UTF-8**                       |
| `List`      | ordered collection of values                  |
| `Map`       | keyed collection of values                    |
| `Structure` | composite value with a type signature         |

**NOTE:** Neither unsigned integers nor 32-bit floating point numbers are included.
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

| Range Minimum              |  Range Maximum             | Optimal Representation
|----------------------------|----------------------------|------------------------
| -9 223 372 036 854 775 808 |             -2 147 483 649 | `INT_64`
|             -2 147 483 648 |                    -32 769 | `INT_32`
|                    -32 768 |                       -129 | `INT_16`
|                       -128 |                        -17 | `INT_8`
|                        -16 |                       +127 | `TINY_INT`
|                       +128 |                    +32 767 | `INT_16`
|                    +32 768 |             +2 147 483 647 | `INT_32`
|             +2 147 483 648 | +9 223 372 036 854 775 807 | `INT_64`


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

| Marker  | Size           |
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

| Marker  | Size                                | Maximum number of bytes |
|---------|-------------------------------------|-------------------------|
| `D0`    | 8-bit big-endian unsigned integer   | 255 bytes               |
| `D1`    | 16-bit big-endian unsigned integer  | 65 535 bytes            |
| `D2     | 32-bit big-endian signed integer    | 2 147 483 648 bytes     |


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
| `String("ABCDEFGHIJKLMNOPQRSTUVWXYZ")       | `D0 1A 41 42 43 44 45 46 47 48 49 4A 4B 4C 4D 4E 4F 50 51 52 53 54 55 56 57 58 59 5A`  |
| `String("Größenmaßstäbe")`                  | `D0 12 47 72 C3 B6 C3 9F 65 6E 6D 61 C3 9F 73 74 C3 A4 62 65`                          |


## List


## Map


## Structure
