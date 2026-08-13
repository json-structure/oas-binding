# An Apache Avro Binding for OpenAPI Descriptions

**Status:** Experimental. Not a submission, and not endorsed by the Apache
Software Foundation or the Apache Avro project. The dialect URI in this
document is a placeholder ([Avro Dialect URI](#avro-dialect-uri)). One of the
media types it uses, `avro/binary`, does not conform to BCP 13
([Registration and Conformance](#registration-and-conformance)).

**Baseline:** Apache Avro Specification 1.12.0 [AVRO], OpenAPI Specification
3.2.0 [OAS].

# Introduction

This document is a dialect binding, in the sense of "Dialect Binding
Requirements" of [BINDING], for Apache Avro schema declarations [AVRO]. It
supplies the declarations of "Binding Parameters" and states the requirements
specific to Avro.

Everything in Part I of [BINDING] applies unchanged and is not restated here.
This document is normative only where it declares a binding parameter or
states an Avro-specific requirement.

An **Avro Schema Object** is a Schema Object whose effective dialect, per
"Dialect Selection" of [BINDING], is the Avro dialect URI.

# Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in BCP 14 [RFC2119] [RFC8174] when, and only when,
they appear in all capitals, as shown here.

**Fullname**, **named type**, **namespace**, and **logical type** are used as
defined in [AVRO].

# Avro Binding Parameters

| Parameter | Value for Avro |
| ---- | ---- |
| Dialect URIs | The single URI in [Avro Dialect URI](#avro-dialect-uri). No vocabularies or add-ins. Logical types are part of the dialect and are always active. |
| Identity keyword | None. Avro defines no identity keyword, and resources are addressable only by location ([Resource Identity](#resource-identity)). |
| Identity comparison | Not applicable. |
| Cross-document mechanism | None. An Avro schema resource MUST be self-contained ([Reference Layers](#reference-layers)). |
| Type determination | [Type Determination for Non-JSON Serializations](#type-determination-for-non-json-serializations). |
| Data media types | Four framings, in [Encodings and Media Types](#encodings-and-media-types). Avro defines a wire encoding, so an Avro-framed body is decoded by the Avro codec and not by the type-determination procedure. |
| Type naming and scope | A named type is named by its fullname. A declared namespace is binding; an undeclared one is materialized from the schema resource ([Type Naming and Scope](#type-naming-and-scope)). |
| Self-description | Every named declaration MUST carry an explicit `namespace` on extraction or aggregation, whether it was declared, inherited, or absent ([Materializing Namespaces](#materializing-namespaces)). |
| Usable schema forms | The JSON object form only ([Usable Schema Forms](#usable-schema-forms)). |

# Avro Dialect URI

The Avro dialect is selected by the URI:

~~~
https://example.org/dialects/avro/1.12#
~~~

Avro defines no meta-schema and no dialect URI. It defines no `$schema`
keyword to carry one. The URI above is a placeholder for the purposes of this
document. A production binding requires a URI minted under the control of
whoever governs the dialect, and no such URI has been minted.

Recognition of this URI is subject to "Recognizing and Rejecting Dialects" of
[BINDING] unchanged. The binding declares no normalization, so matching is
byte-exact. Avro has no derived-meta-schema mechanism, so the derived-URI rule
of that section does not apply.

# Usable Schema Forms

An Avro schema is one of three JSON values [AVRO]:

| Avro form | Example |
| ---- | ---- |
| JSON object | `{"type": "record", "name": "Pet", ...}` |
| JSON string | `"string"`, or a named-type reference such as `"com.example.Pet"` |
| JSON array | a union, such as `["null", "string"]` |

A Schema Object position accepts neither a JSON string nor a JSON array
("Dialect Selection" of [BINDING]). This binding therefore declares the JSON
object form as the only form usable at a Schema Object position.

A conforming Avro Schema Object MUST be a JSON object. Tooling MUST reject a
Description that places a string-form or array-form Avro schema at a Schema
Object position.

An author whose intended resource is a union or a bare primitive MUST wrap it.
A union is wrapped in a record with a single field, or in a `map` or `array`
whose value type is the union. A primitive is written in its equivalent object
form, as `{"type": "string"}`.

The restriction applies to the Schema Object position alone. Inside an Avro
Schema Object, all three forms are unrestricted. A field type MAY be
`"string"`, `["null", "string"]`, or `"com.example.Pet"`.

# Dialect Selection and Self-Description

Selection follows "Dialect Selection" of [BINDING] unchanged.

Avro defines no `$schema` keyword. Avro does permit attributes it does not
define, as metadata that must not affect the serialized form [AVRO]. A
`$schema` attribute at the root of an Avro Schema Object is therefore valid
Avro, and is inert to Avro tooling.

An Avro Schema Object SHOULD carry `$schema` explicitly, so that it remains
self-selecting when extracted from the Description. Where it does not, the
OpenAPI Object's `jsonSchemaDialect` selects the dialect.

Tooling MUST NOT treat a `$schema` attribute as part of the Avro type
declaration, and MUST NOT emit it into any Avro serialized form.

# Resource Identity

This binding declares no identity keyword. "Default Resource Identity" of
[BINDING] therefore does not apply.

An Avro schema resource in an OpenAPI Description is addressable by its OAS
location, and by nothing else. Tooling MUST NOT synthesize a URI identity and
write it into an Avro Schema Object.

There is no identity keyword, so an extracted Avro Schema Object carries
no identity. "Materializing Defaults for Standalone Processing" of [BINDING]
reduces, for this binding, to `$schema` plus the namespace materialization of
[Materializing Namespaces](#materializing-namespaces).

# Reference Layers

The two layers of "Reference Layer Separation" of [BINDING] are instantiated
as follows.

The OpenAPI layer is unchanged. An OAS `$ref` targeting an Avro Schema Object
as a whole resolves per [OAS].

The dialect layer is Avro's named-type reference, being a JSON string in a
schema position that names a type by fullname. It resolves only within the
schema resource that contains it, subject to Avro's rule that a name is
defined before it is used, in depth-first left-to-right traversal order
[AVRO].

This binding declares no cross-document mechanism. "Resolving Cross-Document
References" of [BINDING] therefore does not apply, and neither do its
retrieval, cycle, and graph-limit requirements.

Consequently:

* An Avro Schema Object MUST define every named type it references. Tooling
  MUST reject an Avro Schema Object containing a named-type reference that
  resolves to no definition within that same resource.
* Two `components.schemas` entries that each need a given type MUST each
  define it.
* An Avro field type MUST NOT be an OAS Reference Object. A `$ref` at a field
  type position is not an Avro schema.

Reuse across schema resources is available only at the OpenAPI layer, through
an OAS `$ref` targeting a whole Schema Object. A type needed at a position
nested inside another Avro Schema Object cannot be reached that way and MUST
be defined locally.

# Type Naming and Scope

This section supplies the type naming and scope declaration required by "Type
Identity and Scope" of [BINDING].

A named type is named by its fullname. `record`, `enum`, and `fixed` are the
named types. The anonymous forms, being `array`, `map`, `union`, and the
primitives, name nothing, and the requirements of this section do not apply
to them.

## Declared and Undeclared Namespaces

A declaration's namespace is **declared** where the declaration itself or an
enclosing named schema supplies it, through a `namespace` attribute or a
dotted `name`. The empty string is a `namespace` attribute and declares the
null namespace [AVRO].

A declaration's namespace is **undeclared** where nothing in that chain
supplies one. Avro resolves such a declaration to the null namespace.

The two cases are subject to different rules, stated in the two sections
below.

## Declared Namespaces

Where a namespace is declared, scope is author-declared and is not derived
from the schema resource. A single Avro schema resource MAY populate several
namespaces. Two schema resources of one Description MAY populate the same
namespace.

Tooling MUST preserve a declared namespace exactly. Tooling MUST NOT rewrite,
prefix, or qualify it.

## Undeclared Namespaces

Every undeclared declaration in every schema resource of a Description
resolves to the same null namespace. Two `components.schemas` entries that
each define a record named `Pet`, neither declaring a namespace, both hold the
fullname `Pet`. The Description states nothing about whether the two are the
same type.

Tooling MUST materialize a namespace for every undeclared declaration before
aggregating, per [Materializing Namespaces](#materializing-namespaces). The
materialized namespace is derived from the schema resource.

An Avro Schema Object SHOULD declare a namespace. Under materialization, a
type's identity follows the schema's position in the Description, and moving
a schema resource changes the type.

## Collisions After Materialization

The rules below apply to fullnames as they stand once
[Materializing Namespaces](#materializing-namespaces) has been applied. A
collision that survives materialization is a collision between declared
namespaces.

Where two declarations across schema resources share a fullname and are
identical, the aggregate holds one declaration. A fullname has at most one
definition [AVRO].

Where two declarations across schema resources share a fullname and differ,
tooling MUST report a diagnostic identifying both. Tooling MUST NOT rename
either declaration. Tooling MUST NOT select one and discard the other. Tooling
MUST NOT synthesize a disambiguating namespace. A synthesized namespace
changes a fullname that consumers of the original resource already depend on.

# Materializing Namespaces

This section supplies the self-description declaration required by
"Materializing Defaults for Standalone Processing" of [BINDING].

Avro determines a named type's fullname in three ways [AVRO]:

1. `name` contains a dot. `name` is then the fullname, and any `namespace` is
   ignored.
2. `name` and `namespace` are both present and `name` contains no dot. The
   fullname is the two joined by a dot.
3. `name` is present without `namespace` and contains no dot. The namespace is
   then taken from the most tightly enclosing named schema, or is the null
   namespace if there is none.

Case 3 makes a declaration's identity depend on its surroundings. Extracting
such a declaration, or aggregating it with others, changes what its fullname
resolves against.

On extraction or aggregation, every named declaration MUST carry an explicit
`namespace`. Tooling MUST compute that value as follows:

* Where the declaration's namespace is declared
  ([Declared and Undeclared Namespaces](#declared-and-undeclared-namespaces)),
  the materialized value is the namespace in effect at that declaration's
  position in the originating document.
* Where the declaration's namespace is undeclared, the materialized value is
  the resource namespace of the schema resource that contains it.

Tooling MUST NOT hand an extracted Avro declaration to an Avro tool with a
namespace left implicit.

## Deriving the Resource Namespace

A resource namespace is a valid Avro namespace. Each of its dot-separated
parts MUST start with `[A-Za-z_]` and MUST subsequently contain only
`[A-Za-z0-9_]` [AVRO]. OAS component keys are less restrictive, being
`^[a-zA-Z0-9\.\-_]+$` [OAS]. A key containing a hyphen, or beginning with a
digit, is not usable.

Tooling MUST derive the resource namespace in this order:

1. Where the Schema Object is a `components.schemas` entry whose component
   key is a valid Avro namespace, use the component key. A key containing
   dots yields a multi-part namespace.
2. Otherwise, derive it from the identifying parts of the Schema Object's
   position, being the path template, the HTTP method, and the role, reduced
   to characters valid in a name.
3. Where neither produces a valid Avro namespace, tooling MUST reject the
   aggregation and require an explicit `namespace` on the Schema Object.

The derivation MUST be deterministic for the same Description and JSON
Pointer. Where two schema resources derive the same resource namespace,
tooling MUST reject the aggregation.

## Example: Inherited Namespace

In the schema below, `LineItem` is `com.example.sales.LineItem`:

~~~ json
{
  "type": "record",
  "name": "Order",
  "namespace": "com.example.sales",
  "fields": [
    { "name": "line", "type": {
        "type": "record",
        "name": "LineItem",
        "fields": [ { "name": "sku", "type": "string" } ]
    } }
  ]
}
~~~

Extracting the inner record MUST produce:

~~~ json
{
  "$schema": "https://example.org/dialects/avro/1.12#",
  "type": "record",
  "name": "LineItem",
  "namespace": "com.example.sales",
  "fields": [ { "name": "sku", "type": "string" } ]
}
~~~

Extracting it without the `namespace` produces `LineItem` in the null
namespace, which is a different type with the same field list.

## Effect on Encoded Data

The two materialization cases differ in their effect on encoded data.

Materializing a declared namespace does not change the fullname. The Parsing
Canonical Form transformation already replaces short names with fullnames
using the applicable namespaces [AVRO]. The schema as written and the schema
with the namespace written out therefore have the same canonical form and the
same fingerprint.

Materializing a resource namespace changes the fullname, and with it the
canonical form and the fingerprint.

Resource-namespace materialization applies to the aggregate type space and to
code generation. Tooling MUST encode and decode message bodies against the
schema as written, with declared namespaces materialized and undeclared ones
left in the null namespace. Tooling MUST compute fingerprints against that
same form. Tooling MUST NOT put a resource-namespace-materialized schema on
the wire, and MUST NOT compute a fingerprint from one.

Where a Description uses a media type of
[Encodings and Media Types](#encodings-and-media-types) and leaves namespaces
undeclared, the aggregate type space and the wire hold different fullnames for
the same declaration. Declaring namespaces keeps the two the same.

## Parsing Canonical Form

The Parsing Canonical Form transformation keeps only the attributes relevant
to parsing data, which are `type`, `name`, `fields`, `symbols`, `items`,
`values`, and `size`, and strips all others [AVRO]. A `$schema` attribute is
therefore stripped, and adding one does not change a schema's fingerprint.

# Encodings and Media Types

This section supplies the data media types declaration required by
"Dialect-Defined Body Encodings" of [BINDING].

Avro specifies two serialization encodings, binary and JSON [AVRO]. Two
additional framings wrap the binary encoding: single-object encoding and the
Object Container File. The four differ in what they carry alongside the data,
and therefore in whether a body can be decoded from the Description alone.

| Media type | Framing | Carries the schema |
| ---- | ---- | ---- |
| `avro/binary` | Binary encoding of one datum. | No |
| `application/avro+json` | JSON encoding of one datum. | No |
| `application/vnd.apache.avro.single-object` | `C3 01`, an 8-byte little-endian CRC-64-AVRO fingerprint, then the binary encoding of one datum. | Fingerprint only |
| `application/vnd.apache.avro.ocf` | Object Container File: `Obj` and `01`, a metadata map, a 16-byte sync marker, then data blocks. | Yes, in full |

Binary encoding "does not include field names, self-contained information
about the types of individual bytes, nor field or record separators", so
"readers are wholly reliant on the schema used when the data was encoded"
[AVRO]. The JSON encoding also requires the schema. It "does not distinguish
between int and long, float and double, records and maps, enums and strings"
[AVRO].

A body of any of these media types MUST NOT be subjected to the procedure of
[Type Determination for Non-JSON Serializations](#type-determination-for-non-json-serializations).
That procedure serves the OAS positions listed there. An Avro-framed body is
decoded as a whole by the Avro codec.

## Registration and Conformance

Avro has no IANA-registered media type. The four media types above differ in
how far they depart from BCP 13 [RFC6838].

| Media type | Top-level type | Tree | Status |
| ---- | ---- | ---- | ---- |
| `avro/binary` | `avro`, not registered | not applicable | Non-conforming |
| `application/avro+json` | `application` | standards | Unregistered, registrable |
| `application/vnd.apache.avro.single-object` | `application` | vendor | Unregistered, registrable |
| `application/vnd.apache.avro.ocf` | `application` | vendor | Unregistered, registrable |

`avro` is not a registered top-level type. IANA maintains the "Top-Level Media
Types" registry established by [RFC9694], and `avro` does not appear in it.
[RFC6838], Section 4.2.7, states that definition of a new top-level type name
"MUST be done via a Standards Track RFC" and that "no other mechanism can be
used to define additional type names".

`avro/binary` is therefore not a conforming media type, and it is not
registrable as it stands. Conformance would require an IETF Standards Track
RFC defining an `avro` top-level type. That is a substantially higher bar than
registering a subtype, and [RFC9694] sets it deliberately high. [RFC9695],
which defines the `haptics` top-level type, shows what clearing it involves.

[AVRO] specifies `avro/binary` without reference to BCP 13 and without a
registration. This binding uses it because it is the prevailing practice, per
[Conflict with Avro RPC](#conflict-with-avro-rpc). This document records the
non-conformance and does not correct it.

The other three media types conform syntactically. `application` is a
registered top-level type, and [RFC6838], Section 4.2.5, admits subtypes named
after the application that processes the data, subject to registration. The
`+json` structured syntax suffix is registered [RFC6839].
`application/avro+json` is a convention in common use. The two
`vnd.apache.avro` names are provisional names coined by this document in the
vendor tree ([RFC6838], Section 3.2), where the Apache Software Foundation
would be the appropriate registrant.

A deployment that requires conforming media types SHOULD use the `application`
forms and SHOULD NOT use `avro/binary`.

A deployment MAY use other media types for these framings. Where it does, the
Description MUST make the framing unambiguous, and tooling MUST NOT infer a
framing from a media type this binding does not declare.

## Conflict with Avro RPC

[AVRO] specifies `avro/binary` as the HTTP content type for Avro RPC, under
"HTTP as Transport". An Avro RPC body carries a handshake, framed buffers, a
request or response metadata map, a message name or error flag, and then the
parameters or response value.

This binding uses `avro/binary` for a bare binary-encoded datum, which is the
prevailing usage in HTTP APIs that are not Avro RPC. The two payloads share a
media type and are not interchangeable.

Tooling conforming to this binding MUST treat an `avro/binary` body as a bare
datum. It MUST NOT expect an Avro RPC handshake, framing, or call header. An
API that speaks Avro RPC is outside the scope of this binding
([Protocols](#protocols)).

## Writer's Schema and Reader's Schema

Avro decodes with two schemas. The writer's schema is the one the data was
encoded against, and the reader's schema is the one the application expects.
Where they differ, the reader resolves them per the schema resolution rules of
[AVRO].

The media type determines what role the Schema Object plays.

For `avro/binary` and `application/avro+json`, the body carries no schema. The
Schema Object is the writer's schema. Tooling MUST encode a body of these
media types against the Schema Object. Tooling MUST NOT decode such a body
until it has established the writer's schema, either by accepting the Schema
Object as the writer's schema or by obtaining the writer's schema out of band.

For `application/vnd.apache.avro.single-object`, the body carries an 8-byte
fingerprint of the writer's schema in Parsing Canonical Form. Tooling MUST
compare that fingerprint against the fingerprint of the Schema Object's
Parsing Canonical Form. Where they match, the Schema Object is the writer's
schema. Where they do not, tooling MUST NOT decode the body against the Schema
Object, and MUST resolve the writer's schema from the fingerprint out of band.

For `application/vnd.apache.avro.ocf`, the file's `avro.schema` metadata
property carries the writer's schema in full. That schema is authoritative.
Tooling MUST decode the file against the schema the file carries, and MUST NOT
decode it against the Schema Object. Tooling MAY use the Schema Object as the
reader's schema, and MUST report a diagnostic where the two fail to resolve.

For this media type the Schema Object states what the Description expects, and
the file states what the data is.

## Schema Evolution and the Description

A Schema Object under this binding is part of the wire contract. Editing it
changes what is encoded.

A change to a Schema Object is wire-compatible only where the old schema
resolves against the new one under the schema resolution rules of [AVRO].
Adding a field with a `default`, reordering fields, and the documented type
promotions are resolvable. Adding a field without a `default` and removing a
field the reader requires are not.

This binding defines no mechanism for a Description to carry more than one
version of a schema. Where readers and writers may hold different versions, a
Description MUST use `application/vnd.apache.avro.single-object` or
`application/vnd.apache.avro.ocf`, whose framings identify the writer's schema
on the wire. Tooling MUST NOT rely on schema resolution for an `avro/binary`
or `application/avro+json` body. Nothing in such a body identifies which
schema it was written against.

## Sequential Bodies

An Object Container File holds many objects under one schema. A Description
SHOULD describe it with `itemSchema`, whose Schema Object describes one object
in the file ([BINDING], "Schema Inspection for Non-JSON Serializations"). The
Schema Object MUST NOT attempt to describe the container structure, the
metadata map, or the block framing.

The other three media types carry exactly one datum. A Description describes
them with `schema`.

## Compression

An Object Container File MAY compress its blocks, named by the `avro.codec`
metadata property. Implementations are required to support `null` and
`deflate`. The `bzip2`, `snappy`, `xz`, and `zstandard` codecs are optional
[AVRO].

The codec is internal to the file. It is not HTTP `Content-Encoding`, and
tooling MUST NOT set or interpret `Content-Encoding` on its behalf.

## Fields Within a Datum

A `bytes` or `fixed` field inside an Avro datum is encoded inside the Avro
framing. It is not a `multipart` part and carries no media type of its own.
An Encoding Object MUST NOT be used to assign a content type to a field of an
Avro Schema Object.

Where an Avro datum is itself a `multipart` part, that part's `contentType` is
one of the media types of this section.

# Protocols

This binding covers Avro schema declarations. It does not cover Avro protocol
declarations, the `messages` and `errors` constructs, the handshake, message
framing, or the call format [AVRO].

An OpenAPI Description describes the operation layer, as does Avro's protocol
wire format. This binding does not combine the two.

# Type Determination for Non-JSON Serializations

This section supplies the type-determination procedure required by "Schema
Inspection for Non-JSON Serializations" of [BINDING].

It applies at the OAS positions that section lists, being form, multipart, and
text values that carry no type information of their own. It does NOT apply to
a body of one of the media types in
[Encodings and Media Types](#encodings-and-media-types). Such a body is
decoded by the Avro codec as a whole.

The table below is the mapping of Avro's JSON encoding [AVRO]. That encoding
governs the result, and the spelling of the Avro type name does not.

| Avro type | JSON data type |
| ---- | ---- |
| `null` | null |
| `boolean` | boolean |
| `int`, `long` | number |
| `float`, `double` | number |
| `bytes` | string |
| `string` | string |
| `enum` | string |
| `fixed` | string |
| `record` | object |
| `map` | object |
| `array` | array |
| union | undetermined |

Given a starting-point Avro Schema Object, tooling MUST determine the type of
a value as follows. All steps operate on the schema alone.

1. Resolve the starting point to a type definition. Follow a named-type
   reference to its definition within the same schema resource. Resolution is
   by fullname, with the namespace rules of
   [Materializing Namespaces](#materializing-namespaces)
   applied.
2. Locate the value: a named field under `fields`, the item type under
   `items`, or the value type under `values`.
3. Take the located declaration's Avro type and map it through the table
   above.
4. Disregard `logicalType`. A logical type annotates an underlying Avro type
   and does not change the JSON encoding. Tooling MUST map the underlying
   type.
5. A union yields "undetermined".

The procedure never inspects instance data. Avro requires an explicit type at
every position and has no keyword that makes a declaration apply conditionally
to the data.

## Byte-Valued Strings

`bytes` and `fixed` map to string. That string is not arbitrary text. The JSON
encoding of a byte sequence is a string in which Unicode code points 0 to 255
map to unsigned 8-bit byte values 0 to 255 [AVRO].

Tooling MUST apply that mapping in both directions. Tooling MUST NOT
base64-encode a `bytes` or `fixed` value, and MUST NOT emit a code point above
255 for one.

## Numeric Precision

The JSON encoding represents `long` as a JSON number. Generic JSON tooling
that parses numbers as IEEE 754 double-precision values loses precision above
2^53. Tooling MUST preserve the full 64-bit value.

The binary encoding does not have this problem. `int` and `long` are written
as variable-length zig-zag values and are exact [AVRO]. The same Schema Object
is therefore lossless over `avro/binary` and hazardous over
`application/avro+json` for large `long` values.

## Logical Types

The mapping of the underlying type governs. "A logical type is always
serialized using its underlying Avro type so that values are encoded in
exactly the same way as the equivalent Avro type that does not have a
`logicalType` attribute" [AVRO]. Tooling MUST NOT infer a wire type from the
logical type name.

| Declaration | JSON data type |
| ---- | ---- |
| `{"type": "int", "logicalType": "date"}` | number |
| `{"type": "int", "logicalType": "time-millis"}` | number |
| `{"type": "long", "logicalType": "time-micros"}` | number |
| `{"type": "long", "logicalType": "timestamp-millis"}` | number |
| `{"type": "long", "logicalType": "timestamp-micros"}` | number |
| `{"type": "long", "logicalType": "timestamp-nanos"}` | number |
| `{"type": "long", "logicalType": "local-timestamp-millis"}` | number |
| `{"type": "string", "logicalType": "uuid"}` | string |
| `{"type": "fixed", "size": 16, "logicalType": "uuid"}` | string, byte-valued |
| `{"type": "bytes", "logicalType": "decimal"}` | string, byte-valued |
| `{"type": "fixed", "size": 12, "logicalType": "duration"}` | string, byte-valued |

An Avro `date` is a count of days from the epoch, and a `timestamp-millis` is
a count of milliseconds from the epoch. Both are numbers. Tooling MUST NOT
serialize either as a formatted date string in a form field, path segment,
query parameter, or header value.

A `decimal` is the two's-complement unscaled integer in big-endian order,
carried in `bytes` or `fixed` [AVRO]. It is a byte-valued string under the
rule of [Byte-Valued Strings](#byte-valued-strings). Tooling MUST NOT
serialize it as a decimal numeral.

## Unions

Avro's JSON encoding tags union values [AVRO]. A value of union type is
encoded as JSON `null` when its branch is `null`, and otherwise as a JSON
object with one member, whose name is the branch's name and whose value is the
recursively encoded value.

A union therefore yields "undetermined". Tooling MUST report a diagnostic
identifying the value, and MUST NOT serialize it.

Tooling MUST NOT reduce a two-branch union containing `null` to its other
branch. A field of type `["null", "string"]` does not serialize as a bare
string. It serializes as `null` or as `{"string": "..."}`.

# Examples

## An Avro Schema Object

A `components.schemas` entry carrying its own `$schema`, referenced from
elsewhere in the Description by an ordinary OAS `$ref` to
`#/components/schemas/Pet`.

~~~ yaml
components:
  schemas:
    Pet:
      $schema: https://example.org/dialects/avro/1.12#
      type: record
      name: Pet
      namespace: com.example.petstore
      fields:
        - name: id
          type: { type: string, logicalType: uuid }
        - name: name
          type: string
        - name: tag
          type: ["null", "string"]
          default: null
~~~

The `id` field is a string on the wire. The `tag` field is a union and yields
"undetermined" for non-JSON serializations.

## Using `jsonSchemaDialect` as the Document Default

~~~ yaml
openapi: 3.2.0
jsonSchemaDialect: https://example.org/dialects/avro/1.12#
info:
  title: Petstore
  version: "1.0.0"
components:
  schemas:
    Pet:
      type: record
      name: Pet
      namespace: com.example.petstore
      fields:
        - name: id
          type: { type: string, logicalType: uuid }
~~~

Extraction for standalone processing MUST materialize `$schema`. No identity
is materialized, and `Pet` already declares its namespace explicitly, so
nothing else changes.

## A Fullname Collision Across Resources

~~~ yaml
components:
  schemas:
    Pet:
      type: record
      name: Pet
      namespace: com.example.petstore
      fields:
        - name: id
          type: { type: string, logicalType: uuid }
    PetListResponse:
      type: record
      name: PetListResponse
      namespace: com.example.petstore
      fields:
        - name: pets
          type:
            type: array
            items:
              type: record
              name: Pet
              fields:
                - name: id
                  type: { type: string, logicalType: uuid }
                - name: tags
                  type: { type: array, items: string }
~~~

The inner `Pet` declares no namespace and inherits `com.example.petstore` from
`PetListResponse`. Both resources therefore declare
`com.example.petstore.Pet`, and the two declarations differ.

Each resource is valid Avro on its own. Aggregating them is a fullname
collision, and tooling MUST report a diagnostic per
[Type Naming and Scope](#type-naming-and-scope).

The author resolves it in the Description, by giving the inner record its own
namespace or by making the two declarations identical.

## Undeclared Namespaces Across Resources

The same two entries with no `namespace` anywhere.

~~~ yaml
components:
  schemas:
    Pet:
      type: record
      name: Pet
      fields:
        - name: id
          type: { type: string, logicalType: uuid }
    PetListResponse:
      type: record
      name: PetListResponse
      fields:
        - name: pets
          type:
            type: array
            items:
              type: record
              name: Pet
              fields:
                - name: id
                  type: { type: string, logicalType: uuid }
                - name: tags
                  type: { type: array, items: string }
~~~

As written, every declaration is in the null namespace, and both entries hold
the fullname `Pet`. The Description places nothing in a shared scope.

Materialization derives a resource namespace from each component key and
produces four distinct fullnames:

| Resource | Declaration | Fullname after materialization |
| ---- | ---- | ---- |
| `Pet` | root record | `Pet.Pet` |
| `PetListResponse` | root record | `PetListResponse.PetListResponse` |
| `PetListResponse` | nested `Pet` | `PetListResponse.Pet` |

The aggregate holds two distinct pet types. No diagnostic is reported, no
collision having survived materialization.

On the wire, both records remain `Pet` in the null namespace. Their
fingerprints differ, their field lists being different. Declaring namespaces
explicitly keeps the type identity and the wire identity the same.

## Operations Over Each Framing

The same `Pet` schema under three framings. Only the media type changes.

~~~ yaml
paths:
  /pets:
    post:
      requestBody:
        content:
          avro/binary:
            schema:
              $ref: "#/components/schemas/Pet"
          application/avro+json:
            schema:
              $ref: "#/components/schemas/Pet"
      responses:
        "200":
          description: The stored pet, tagged with its writer's schema.
          content:
            application/vnd.apache.avro.single-object:
              schema:
                $ref: "#/components/schemas/Pet"
  /pets/export:
    get:
      responses:
        "200":
          description: All pets as an Avro Object Container File.
          content:
            application/vnd.apache.avro.ocf:
              itemSchema:
                $ref: "#/components/schemas/Pet"
~~~

For the two request media types, `Pet` is the writer's schema. A client
encodes against it. A server that cannot accept `Pet` as the writer's schema
MUST reject the request.

For the `200` response on `/pets`, the body carries a fingerprint. A client
MUST compare it against the fingerprint of `Pet` in Parsing Canonical Form
before decoding against `Pet`.

For `/pets/export`, the file carries `avro.schema`, and that schema decodes
the file. `Pet` is the reader's schema. `itemSchema` marks it as describing
one object in the file, not the container.

# Conformance

A conforming Avro Schema Object is a Schema Object that meets three
conditions. Its effective dialect is the URI of
[Avro Dialect URI](#avro-dialect-uri). It is a JSON object
([Usable Schema Forms](#usable-schema-forms)). Its content, disregarding any
`$schema` attribute, is a valid Avro schema declaration that defines every
named type it references.

The roles of "Conformance" in [BINDING] apply. Two are narrowed by this
binding's declarations:

Resolver:
: There is no cross-document mechanism to resolve. A conforming Resolver for
  this binding resolves named-type references within a schema resource, and
  MUST reject a reference that resolves to no definition there.

Codec / Code Generator:
: A conforming Codec or Code Generator MUST use the procedure of
  [Type Determination for Non-JSON Serializations](#type-determination-for-non-json-serializations)
  and MUST NOT fall back to the OAS-dialect procedure. A Codec that encodes or
  decodes a body of a media type in
  [Encodings and Media Types](#encodings-and-media-types) MUST establish the
  writer's schema per
  [Writer's Schema and Reader's Schema](#writers-schema-and-readers-schema)
  before decoding, and MUST NOT decode against an unverified schema.

# Security Considerations

The cross-document retrieval, external-resource, and import-graph
considerations of [BINDING] do not apply. This binding declares no
cross-document mechanism, so an Avro Schema Object cannot cause a retrieval.

Dialect confusion:
: Avro and the OAS dialect share the `type` keyword and several of its
  values. `{"type": "string"}` means the same in both. Others diverge
  silently. `{"type": "string", "logicalType": "uuid"}` constrains the value
  to a UUID in Avro, and constrains nothing beyond `string` under the OAS
  dialect, which does not define `logicalType`. Tooling MUST NOT process an
  Avro Schema Object as an OAS-dialect schema, or the reverse.

Schemas carried in a message body:
: An Object Container File carries a writer's schema in its `avro.schema`
  metadata property. Decoding the file means parsing a schema supplied by
  whoever produced the body. That schema is untrusted input, and it drives
  allocation. Tooling MUST validate it as an Avro schema declaration before
  use. Tooling MUST bound what it will accept from it, at least on the size of
  a `fixed`, the depth of nesting, the number of named types, and the total
  size of the schema text. Tooling MUST NOT allocate from a declared size
  before the corresponding bytes have been read.

Block counts and sizes:
: Avro encodes arrays and maps as blocks, each introduced by a `long` count
  and, where the count is negative, a `long` byte size [AVRO]. An Object
  Container File likewise introduces each data block with a count and a size.
  These values come from the wire and can be arbitrarily large. Tooling MUST
  bound them against the bytes actually available, and MUST NOT pre-allocate
  on the strength of a declared count or size.

Decompression:
: An Object Container File names its block codec in `avro.codec`, and the
  optional codecs include `bzip2`, `snappy`, `xz`, and `zstandard` [AVRO].
  Tooling MUST enforce a bound on the decompressed size of a block and on the
  decompression ratio. Tooling MUST reject a codec it has not been configured
  to accept, and MUST NOT treat an unrecognized `avro.codec` value as `null`.

Schema fingerprints:
: The single-object framing identifies the writer's schema by a 64-bit
  CRC-64-AVRO fingerprint. [AVRO] states that Avro fingerprints "are not meant
  to provide any security guarantees" and recommends that surrounding
  mechanisms prevent collision and pre-image attacks. Tooling MUST NOT treat a
  fingerprint match as authentication of the body, and MUST NOT resolve a
  fingerprint against a schema source it does not trust.

Untrusted schemas:
: An Avro schema obtained from an external source is untrusted input. Tooling
  MUST validate it as an Avro schema declaration before use.

Recursive definitions:
: Avro named types may be mutually recursive within one schema resource.
  Tooling that expands a schema into an in-memory type or a validator MUST
  detect recursion and MUST NOT expand indefinitely.

Otherwise, the security considerations of [OAS] and [BINDING] apply
unchanged.

# References

**[AVRO]** Apache Software Foundation, "Apache Avro Specification", version
1.12.0. <https://avro.apache.org/docs/1.12.0/specification/>

**[BINDING]** Vasters, C., "Binding JSON Structure to the OpenAPI
Specification", [draft-vasters-json-structure-oas-binding](draft-vasters-json-structure-oas-binding.md).

**[OAS]** OpenAPI Initiative, "OpenAPI Specification", version 3.2.0.
<https://spec.openapis.org/oas/v3.2.0.html>

**[RFC2119]** Bradner, S., "Key words for use in RFCs to Indicate Requirement
Levels", BCP 14, RFC 2119, March 1997.

**[RFC6838]** Freed, N., Klensin, J., and T. Hansen, "Media Type
Specifications and Registration Procedures", BCP 13, RFC 6838, January 2013.

**[RFC6839]** Hansen, T. and A. Melnikov, "Additional Media Type Structured
Syntax Suffixes", RFC 6839, January 2013.

**[RFC8174]** Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key
Words", BCP 14, RFC 8174, May 2017.

**[RFC9694]** Dürst, M., "Guidelines for the Definition of New Top-Level Media
Types", BCP 13, RFC 9694, March 2025.

**[RFC9695]** Muthusamy, Y. and C. Ullrich, "The 'haptics' Top-Level Media
Type", RFC 9695, March 2025.
