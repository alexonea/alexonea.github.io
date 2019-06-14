---
layout: post
title: "GUID validation at compile time using C++11"
date: 2019-06-06
---

Globally Unique Identifiers (GUIDs), sometimes referred as Universaly Unique Identifiers (UUIDs), are 128-bit reference numbers used to identify and version objects which are meant to be shared. Examples of GUID usage are: primary keys in databases, unique file names and unique interface identifiers in interface programming (e.g. Microsoft COM) technologies.

This post discusses a possible implementation of compile-time validation and parsing of GUIDs using C++11.

# Introduction
Regardless of their use in our programs, GUIDs are, by their nature, constant. In some applications, GUIDs are generated outside of the source code and they are associated with objects in our programs for identification and/or versioning. In such cases, the GUIDs appear in source code as constant definitions, either in ASCII form or as binary data.

# GUID representation

For demonstration purposes, the following structure has been chosen to represent a GUID object:
```c++
struct GUID
{
    uint32_t data1;
    uint16_t data2;
    uint16_t data3;
    uint8_t  data4[8];
};
```

# Requirements

We wish to have a function or macro that we can use in a way that it converts a GUID from ASCII form to its binary representation, something similar to this:
```c++
GUID myGuid = parseGUID ("11223344-5566-7788-9900-AABBCCDDEEFF");
```
In addition, we wish that the parsing is done at compile time and that `myGuid` is placed in memory as if it was defined in the following way:
```c++
GUID refGUID =
{
    0x11223344,
    0x5566,
    0x7788,
    {
        0x99,
        0x00,
        0xAA,
        0xBB,
        0xCC,
        0xDD,
        0xEE,
        0xFF,
    },
};
```

Therefore, we summarize the requirements for our `parseGUID` function as follows:
1. return a binary representation of ASCII input sequence in case it represents a valid GUID
2. throw a compile-time error or run-time exception in case of ill-formed ASCII input sequences (that do not represent valid GUIDs)
3. perform the parsing at compile time, where applicable
4. Allow any combination of dashes and spaces in the ASCII input format, as well as a leading and trailling `{}`.

To check that the first requirement is fulfilled, we will compare the memory contents of a reference GUID with the one parsed using our magic function `parseGUID`:
```c++
assert (0 == memcmp (&myGUID, &refGUID, sizeof (GUID))); // should be true
```

# Parsing a single character
We start by thinking of how to parse at compile time a single character and give of its binary value. Let's consider a function `__parseU8` which takes as input an `uint8_t` single character and returns the value it represents. A naive first attempt would be:
```c++
uint64_t
__parseU8 (uint8_t c)
{
    if (c >= '0' && c <= '9') return c - '0';
    if (c >= 'A' && c <= 'F') return c - 'A' + 10;
    if (C >= 'a' && c <= 'f') return c - 'a' + 10;

    throw std::runtime_error ("Illegal character");
}
```

We take advantage of the fact that characters representing digits and letters have consecutive ASCII representations, therefore we can return their offsets in relation to the first character in their respective series. For letters `A-Za-z` we also offset the value by 10 since they represent values in the range `[10-16]` in hex.
