# JSON Event Format for CloudEvents - Version 0.4-wip

## Abstract

The JSON Format for CloudEvents defines how events are expressed in JavaScript
Object Notation (JSON) Data Interchange Format ([RFC8259][rfc8259]).

## Status of this document

This document is a working draft.

## Table of Contents

1. [Introduction](#1-introduction)
2. [Attributes](#2-attributes)
3. [Envelope](#3-envelope)
4. [JSON Batch Format](#4-json-batch-format)
5. [References](#5-references)

## 1. Introduction

[CloudEvents][ce] is a standardized and transport-neutral definition of the
structure and metadata description of events. This specification defines how the
elements defined in the CloudEvents specification are to be represented in the
JavaScript Object Notation (JSON) Data Interchange Format ([RFC8259][rfc8259]).

The [Attributes](#2-attributes) section describes the naming conventions and
data type mappings for CloudEvents attributes.

The [Envelope](#3-envelope) section defines a JSON container for CloudEvents
attributes and an associated media type.

### 1.1. Conformance

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in [RFC2119][rfc2119].

## 2. Attributes

This section defines how CloudEvents attributes are mapped to JSON. This
specification does not explicitly map each attribute, but provides a generic
mapping model that applies to all current and future CloudEvents attributes,
including extensions.

For clarity, extension attributes are serialized using the same rules as
specification defined attributes. This includes their syntax and placement
within the JSON object. In particular, extensions are placed as top-level JSON
properties. Extensions themselves are free to have nested properties, however
the root of the extension MUST be serialized as a top-level JSON property. There
were many reason for this design decision and they are covered in more detail in
the [Primer](primer.md#json-extensions).

### 2.1. Base Type System

The core [CloudEvents specification][ce] defines a minimal abstract type system,
which this mapping leans on.

### 2.2. Type System Mapping

The [CloudEvents type system][ce-types] MUST be mapped to JSON types as follows,
with exceptions noted below.

| CloudEvents   | JSON                                                           |
| ------------- | -------------------------------------------------------------- |
| String        | [string][json-string]                                          |
| Integer       | [number][json-number], only the `int` component is permitted   |
| Binary        | [string][json-string], [Base64-encoded][base64] binary         |
| URI-reference | [string][json-string] following [RFC 3986][rfc3986]            |
| Timestamp     | [string][json-string] following [RFC 3339][rfc3339] (ISO 8601) |
| Map           | [JSON object][json-object]                                     |
| Any           | [JSON value][json-value]                                       |

Extension specifications MAY define diverging mapping rules for the values of
attributes they define.

For instance, the attribute value might be a data structure defined in a
standard outside of CloudEvents, with a formal JSON mapping, and there might be
risk of translation errors or information loss when the original format is not
preserved.

An extension specification that defines a diverging mapping rule for JSON, and
any revision of such a specification, MUST also define explicit mapping rules
for all other event formats that are part of the CloudEvents core at the time of
the submission or revision.

If required, like when decoding Maps, the CloudEvents type can be determined by
inference using the rules from the mapping table, whereby the only potentially
ambiguous JSON data type is `string`. The value is compatible with the
respective CloudEvents type when the mapping rules are fulfilled.

### 2.3. Mapping Data

If an implementation determines that the actual type of `data` is a `String`,
the value MUST be represented as [JSON string][json-string] expression; for
`Binary`, the value MUST represented as [JSON string][json-string] expression
containing the [Base64][base64] encoded binary value.

### 2.4. Examples

The following table shows exemplary mappings:

| CloudEvents     | Type          | Exemplary JSON Value            |
| --------------- | ------------- | ------------------------------- |
| type            | String        | "com.example.someevent"         |
| specversion     | String        | "0.4-wip"                       |
| source          | URI-reference | "/mycontext"                    |
| id              | String        | "1234-1234-1234"                |
| time            | Timestamp     | "2018-04-05T17:31:00Z"          |
| datacontenttype | String        | "application/json"              |
| data            | String        | "<much wow=\"xml\"/>"           |
| data            | Binary        | "Q2xvdWRFdmVudHM="              |
| data            | Map           | { "objA" : "vA", "objB", "vB" } |

### 2.5. JSONSchema Validation

The CloudEvents [JSONSchema](http://json-schema.org) for the spec is located
[here](spec.json) and contains the definitions for validating events in JSON.

## 3. Envelope

Each CloudEvents event can be wholly represented as a JSON object.

Such a representation MUST use the media type `application/cloudevents+json`.

All REQUIRED and all not omitted OPTIONAL attributes in the given event MUST
become members of the JSON object, with the respective JSON object member name
matching the attribute name, and the member's type and value being mapped using
the [type system mapping](#22-type-system-mapping).

### 3.1. Special Handling of "data"

The mapping of `data` follows the rules laid out in
[Section 2.3.](#23-mapping-data), with two additional rules:

First, if an implementation determines that the type of `data` is
`Binary` or `String`, it MUST inspect the `datacontenttype` attribute to
determine whether it is indicated that the data value contains JSON data.

If the `datacontenttype` value is either ["application/json"][rfc4627] or any
media type with a [structured +json suffix][rfc6839], the implementation MUST
translate the `data` value into a [JSON value][json-value], and set
the `data` attribute of the envelope JSON object to this JSON value.

If the `datacontenttype` value does not follow the [structured +json
suffix][rfc6839] but is known to use JSON encoding, the implementation MUST
translate the `data` value into a [JSON value][json-value], and set
the `data` attribute of the envelope JSON object to this JSON value. Its typical
examples are, but not limited to, `text/json`,
[`application/json-seq`][json-seq] and
[`application/geo+json-seq`][json-geoseq].

Unlike all other attributes, for which value types are restricted to strings per
the [type-system mapping](#22-type-system-mapping), the resulting `data` member
[JSON value][json-value] is unrestricted, and MAY also contain numeric and
logical JSON types.

Second, whether a Base64-encoded string in `data` is treated as
`Binary` or as a `String` is also determined by the `datacontenttype` value. If
the `datacontenttype` media type is known to contain text, the data attribute
value is not further interpreted and treated as a text string. Otherwise, it is
decoded and treated as a binary value.

### 3.2. Examples

Example event with `String`-valued `data`:

```JSON
{
    "specversion" : "0.4-wip",
    "type" : "com.example.someevent",
    "source" : "/mycontext",
    "id" : "A234-1234-1234",
    "time" : "2018-04-05T17:31:00Z",
    "comexampleextension1" : "value",
    "comexampleextension2" : {
        "otherValue": 5
    },
    "datacontenttype" : "text/xml",
    "data" : "<much wow=\"xml\"/>"
}
```

Example event with `Binary`-valued data

```JSON
{
    "specversion" : "0.4-wip",
    "type" : "com.example.someevent",
    "source" : "/mycontext",
    "id" : "B234-1234-1234",
    "time" : "2018-04-05T17:31:00Z",
    "comexampleextension1" : "value",
    "comexampleextension2" : {
        "otherValue": 5
    },
    "datacontenttype" : "application/vnd.apache.thrift.binary",
    "data" : "... base64 encoded string ..."
}
```

Example event with JSON data for the "data" member, either derived from a `Map`
or [JSON data](#31-special-handling-of-data) data:

```JSON
{
    "specversion" : "0.4-wip",
    "type" : "com.example.someevent",
    "source" : "/mycontext",
    "id" : "C234-1234-1234",
    "time" : "2018-04-05T17:31:00Z",
    "comexampleextension1" : "value",
    "comexampleextension2" : {
        "otherValue": 5
    },
    "datacontenttype" : "application/json",
    "data" : {
        "appinfoA" : "abc",
        "appinfoB" : 123,
        "appinfoC" : true
    }
}
```

## 4. JSON Batch Format

In the _JSON Batch Format_ several CloudEvents are batched into a single JSON
document. The document is a JSON array filled with CloudEvents in the [JSON
Event format][json-format].

Although the _JSON Batch Format_ builds ontop of the _JSON Format_, it is
considered as a separate format: a valid implementation of the _JSON Format_
doesn't need to support it. The _JSON Batch Format_ MUST NOT be used when only
support for the _JSON Format_ is indicated.

### 4.1. Mapping CloudEvents

This section defines how a batch of CloudEvents is mapped to JSON.

The outermost JSON element is a [JSON Array][json-array], which contains as
elements CloudEvents rendered in accordance with the [JSON event
format][json-format] specification.

### 4.2. Envelope

A JSON Batch of CloudEvents MUST use the media type
`application/cloudevents-batch+json`.

### 4.3. Examples

An example containing two CloudEvents: The first with `Binary`-valued data, the
second with JSON data.

```JSON
[
  {
      "specversion" : "0.4-wip",
      "type" : "com.example.someevent",
      "source" : "/mycontext/4",
      "id" : "B234-1234-1234",
      "time" : "2018-04-05T17:31:00Z",
      "comexampleextension1" : "value",
      "comexampleextension2" : {
          "otherValue": 5
      },
      "datacontenttype" : "application/vnd.apache.thrift.binary",
      "data" : "... base64 encoded string ..."
  },
  {
      "specversion" : "0.4-wip",
      "type" : "com.example.someotherevent",
      "source" : "/mycontext/9",
      "id" : "C234-1234-1234",
      "time" : "2018-04-05T17:31:05Z",
      "comexampleextension1" : "value",
      "comexampleextension2" : {
          "otherValue": 5
      },
      "datacontenttype" : "application/json",
      "data" : {
          "appinfoA" : "abc",
          "appinfoB" : 123,
          "appinfoC" : true
      }
  }
]
```

An example of an empty batch of CloudEvents (typically used in a response, but
also valid in a request):

```JSON
[]
```

## 5. References

- [RFC2046][rfc2046] Multipurpose Internet Mail Extensions (MIME) Part Two:
  Media Types
- [RFC2119][rfc2119] Key words for use in RFCs to Indicate Requirement Levels
- [RFC4627][rfc4627] The application/json Media Type for JavaScript Object
  Notation (JSON)
- [RFC4648][rfc4648] The Base16, Base32, and Base64 Data Encodings
- [RFC6839][rfc6839] Additional Media Type Structured Syntax Suffixes
- [RFC8259][rfc8259] The JavaScript Object Notation (JSON) Data Interchange
  Format

[base64]: https://tools.ietf.org/html/rfc4648#section-4
[ce]: ./spec.md
[ce-types]: ./spec.md#type-system
[content-type]: https://tools.ietf.org/html/rfc7231#section-3.1.1.5
[json-format]: ./json-format.md
[json-geoseq]:
  https://www.iana.org/assignments/media-types/application/geo+json-seq
[json-object]: https://tools.ietf.org/html/rfc7159#section-4
[json-seq]: https://www.iana.org/assignments/media-types/application/json-seq
[json-number]: https://tools.ietf.org/html/rfc7159#section-6
[json-string]: https://tools.ietf.org/html/rfc7159#section-7
[json-value]: https://tools.ietf.org/html/rfc7159#section-3
[json-array]: https://tools.ietf.org/html/rfc7159#section-5
[rfc2046]: https://tools.ietf.org/html/rfc2046
[rfc2119]: https://tools.ietf.org/html/rfc2119
[rfc3986]: https://tools.ietf.org/html/rfc3986
[rfc4627]: https://tools.ietf.org/html/rfc4627
[rfc4648]: https://tools.ietf.org/html/rfc4648
[rfc6839]: https://tools.ietf.org/html/rfc6839#section-3.1
[rfc8259]: https://tools.ietf.org/html/rfc8259
[rfc3339]: https://www.ietf.org/rfc/rfc3339.txt
