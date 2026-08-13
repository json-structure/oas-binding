# An Apache Avro Binding for OpenAPI Descriptions

**Status:** Experimental. Not a submission, and not endorsed by the Apache
Software Foundation or the Apache Avro project. The dialect URI in this
document is a placeholder ([Avro Dialect URI](#avro-dialect-uri)).

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
| Type naming and scope | A named type is named by its fullname. Scope is author-declared: it is the `namespace` in effect at the declaration ([Type Naming and Scope](#type-naming-and-scope)). |
| Self-description | An inner named declaration inherits its namespace from the most tightly enclosing named schema. That value MUST be materialized on extraction ([Materializing the Inherited Namespace](#materializing-the-inherited-namespace)). |
| Usable schema forms | The JSON object form only ([Usable Schema Forms](#usable-schema-forms)). |

# Avro Dialect URI

The Avro dialect is selected by the URI:

~~~
https://example.org/dialects/avro/1.12#
~~~

Avro defines no meta-schema and no dialect URI, because it defines no
`$schema` keyword to carry one. The URI above is a placeholder for the
purposes of this document. A production binding requires a URI minted under
the control of whoever governs the dialect. No such URI exists.

Recognition of this URI is subject to "Recognizing and Rejecting Dialects" of
[BINDING] unchanged. The binding declares no normalization, so matching is
byte-exact. Avro has no derived-meta-schema mechanism, so the derived-URI rule
of that section has nothing to apply to.

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

Because there is no identity keyword, an extracted Avro Schema Object carries
no identity. "Materializing Defaults for Standalone Processing" of [BINDING]
reduces, for this binding, to `$schema` plus the namespace materialization of
[Materializing the Inherited Namespace](#materializing-the-inherited-namespace).

# Reference Layers

The two layers of "Reference Layer Separation" of [BINDING] are instantiated
as follows.

The OpenAPI layer is unchanged. An OAS `$ref` targeting an Avro Schema Object
as a whole resolves per [OAS].

The dialect layer is Avro's named-type reference: a JSON string in a schema
position that names a type by fullname. It resolves only within the schema
resource that contains it, subject to Avro's rule that a name is defined
before it is used, in depth-first left-to-right traversal order [AVRO].

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
  type position is not an Avro schema, and Avro's metadata allowance does not
  make it one.

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

Scope is author-declared, not derived from the schema resource. The scope of
a named declaration is the `namespace` in effect at that declaration. A single
Avro schema resource MAY populate several namespaces, and two schema resources
of one Description MAY populate the same namespace.

Aggregating the Avro Schema Objects of a Description into one type space
therefore places two declarations in one scope exactly when their fullnames
agree. That is the author's own arrangement, and preserving it satisfies the
aggregation requirement of [BINDING].

Where two declarations across schema resources share a fullname and are
identical, the aggregate holds one declaration. Avro admits no other outcome,
since a fullname has at most one definition [AVRO].

Where two declarations across schema resources share a fullname and differ,
tooling MUST report a diagnostic identifying both. Tooling MUST NOT rename
either declaration. Tooling MUST NOT select one and discard the other. Tooling
MUST NOT synthesize a disambiguating namespace, because a synthesized
namespace changes a fullname that consumers of the original resource already
depend on.

# Materializing the Inherited Namespace

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

The third case makes an inner declaration's identity depend on its container.
Extracting the inner declaration changes its fullname.

Tooling that extracts an Avro Schema Object, or any named declaration nested
within one, MUST first write an explicit `namespace` onto every named
declaration that would otherwise resolve its namespace by case 3. The
materialized value MUST be the namespace in effect at that declaration's
position in the originating document.

Tooling MUST NOT hand an extracted Avro declaration to an Avro tool with an
inherited namespace left implicit.

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

# Type Determination for Non-JSON Serializations

This section supplies the type-determination procedure required by "Schema
Inspection for Non-JSON Serializations" of [BINDING].

Avro defines its own JSON encoding [AVRO]. That encoding governs, and not the
spelling of the Avro type name.

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
   [Materializing the Inherited Namespace](#materializing-the-inherited-namespace)
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

## Logical Types

The mapping of the underlying type governs. Tooling MUST NOT infer a wire type
from the logical type name.

| Declaration | JSON data type |
| ---- | ---- |
| `{"type": "int", "logicalType": "date"}` | number |
| `{"type": "int", "logicalType": "time-millis"}` | number |
| `{"type": "long", "logicalType": "timestamp-millis"}` | number |
| `{"type": "long", "logicalType": "timestamp-micros"}` | number |
| `{"type": "string", "logicalType": "uuid"}` | string |
| `{"type": "bytes", "logicalType": "decimal"}` | string |
| `{"type": "fixed", "size": 12, "logicalType": "duration"}` | string |

An Avro `date` is a count of days from the epoch, and a `timestamp-millis` is
a count of milliseconds from the epoch. Both are numbers. Tooling MUST NOT
serialize either as a formatted date string in a form field, path segment,
query parameter, or header value.

## Unions

Avro's JSON encoding tags union values [AVRO]. A value of union type is
encoded as JSON `null` when its branch is `null`, and otherwise as a JSON
object with one member, whose name is the branch's name and whose value is the
recursively encoded value.

A union therefore yields "undetermined", and tooling MUST report a diagnostic
identifying the value rather than serializing it.

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
  and MUST NOT fall back to the OAS-dialect procedure.

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

**[RFC8174]** Leiba, B., "Ambiguity of Uppercase vs Lowercase in RFC 2119 Key
Words", BCP 14, RFC 8174, May 2017.
