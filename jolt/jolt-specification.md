---
layout: default
---
# Jolt Specification v1 [DRAFT]

Jolt is a scheme for the [JSON](http://json.org/)-based representation of the PackStream type system, as used by Bolt.


## Overview

JSON is only capable of natively representing a subset of PackStream types.
This scheme introduces the ability to represent other types, primarily through the use of _singleton objects_.
The key of the object is used to denote the type, with the value transferred in a lossless way, such as in a JSON string.

Type keys are short strings, often single characters, that represent a particular type.
The minimal size of these strings allows the JSON document to still easily be scanned by eye and reduces the number of duplicate or redundant bytes in a transmission.

The following sections describe the Jolt representations for each PackStream type.
It is expected that the reader is already familiar with the PackStream type system.


## Null

**Null** is always encoded as a JSON `null`.


## Boolean

**Boolean** values are always encoded as JSON `true` and `false`.


## Integer & Float

JSON only supports one type of number.
This gives a potential for overlap and confusion when encoding **Integer** and **Float** values.
For that reason, both types have a singleton object representation which can be used to avoid ambiguity.

### Integer Examples

The representations below should _always_ be interpreted as **Integer** values, with any fractional numbers truncated using `Math.floor`.
```
{"Z": "123"}
{"Z": "0"}
{"Z": "-12345"}
```

### Float Examples

The following representations should always be interpreted as Float values.
Note that special floating point values can also be passed within the singleton object value.
These values are not otherwise available within the official JSON specification, and implementations vary as to their provision for these.

```
{"R": "123"}
{"R": "+0.0"}
{"R": "-0.0"}
{"R": "NaN"}
{"R": "+Infinity"}
{"R": "-Infinity"}
```

### Plain Numbers

Plain numbers can be used but should be encoded and interpreted according to the result of the following function:

```
function isInt32(n) {
    return (Number.isInteger(n) && 
            n >= -0x80000000 && n < 0x80000000);
}
```

This function checks for whole numbers in the int32 range.
If this function returns true, the value should be interpreted as an **Integer**; if false, as a **Float**.
Therefore, `123` would be interpreted as an **Integer**, whereas `123.4` and `0xABCD1234` would be interpreted as a **Float**.

For this reason, it is necessary to always use the longhand encoding for integers outside the int32 range, and similarly for whole number float values within that range.

### Rationale behind 'Z' and 'R'

The sigils `Z` and `R` are taken from the mathematical symbols for the sets of integers and real numbers, ℤ and ℝ.
Z originates from the German word, Zahlen, meaning numbers.


## String

**String** values are always encoded as JSON string values.

### Example

```
"hello, world"
```


## Bytes

**Bytes** are encoded in a singleton object, with the type sigil `#`.
The value is a string of two-character hexadecimal byte values.
Values can be upper or lower case and may contain spaces for human readability.
Spaces may only occur between pairs of characters and should be ignored by parsers.

### Examples

```
{"#": "ABCD0123CDEF4567"}
{"#": "abcd0123cdef4567"}
{"#": "AB CD 01 23 CD EF 45 67"}
```

### Rationale

The sigil `#` and the style of value encoding mirror the hexadecimal notation used in CSS colour codes.


## List

**List** values are always encoded as a JSON array.


## Map

**Map** values are encoded as JSON object values, nested within a singleton object, under the two-character type sigil `{}`.

### Example

```
{"{}": {"name": "Alice", "age": 33}}
```

### Rationale

Since there is no obvious candidate for a single character type sigil, a two character variant was chosen for clarity.


## Structure

**Structure** values are encoded as singleton objects.
The type key starts with a `$` sigil followed by characters that represent the structure subtype code.
The value can be of any type and is implementation-defined.

Bolt permits subtype codes between 0 and 127 inclusive.
For any codes between 33 and 126 (the printable ASCII characters) the type key should be `$` followed by the ASCII character for that code.
For other values, a two-character hex code should be used instead.

### Examples

```
{"$N": [123, ["Person"], {"{}": {"name": "Alice"}}]}
{"$D": "2002-04-16"}
{"$0A": ["foo", "bar"]}
```

### Rationale

The '$' sigil for "Structure" was chosen as it (subjectively) looks similar to an "S" but does not read as one, due to not being part of the alphabet.
This is intended to create a visual separation between the two parts of the key.
For example, "$0D" should be read as "the structure with subtype code 0D" rather than "S0D" which looks like a single word.
The dollar is also a valid identifier character in some languages, such as JavaScript.
