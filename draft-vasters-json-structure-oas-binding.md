---

title: "JSON Structure: OpenAPI Binding"
abbrev: "JSON Structure OAS Binding"
category: std

docname: draft-vasters-json-structure-oas-binding-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date: 2026-08-12
consensus: true
v: 0
area: Web and Internet Transport
workgroup: Building Blocks for HTTP APIs
keyword: Internet-Draft
venue:
  group: TBD
  type: Working Group
  mail: TBD
  arch: TBD
  github: "json-structure/oas-binding"
  latest: "https://json-structure.github.io/oas-binding/draft-vasters-json-structure-oas-binding.html"

author:
 -
    fullname: Clemens Vasters
    organization: Microsoft Corporation
    email: clemensv@microsoft.com

normative:
  RFC2119:
  RFC3986:
  RFC6901:
  RFC8174:
  RFC8259:
  OAS:
    title: "OpenAPI Specification"
    author:
    - org: OpenAPI Initiative
    date: 2025
    target: https://spec.openapis.org/oas/latest
  JSTRUCT-CORE:
    title: "JSON Structure Core"
    author:
      - fullname: Clemens Vasters
    target: https://json-structure.github.io/core/draft-vasters-json-structure-core.html
  JSTRUCT-IMPORT:
    title: "JSON Structure Import"
    author:
      - fullname: Clemens Vasters
    target: https://json-structure.github.io/import/draft-vasters-json-structure-import.html
  JSTRUCT-UNITS:
    title: "JSON Structure: Symbols, Scientific Units, and Currencies"
    author:
      - fullname: Clemens Vasters
    target: https://json-structure.github.io/units/draft-vasters-json-structure-units.html
  JSTRUCT-VALIDATION:
    title: "JSON Structure Validation"
    author:
      - fullname: Clemens Vasters
    target: https://json-structure.github.io/validation/draft-vasters-json-structure-validation.html
  JSTRUCT-COMPOSITION:
    title: "JSON Structure Conditional Composition"
    author:
      - fullname: Clemens Vasters
    target: https://json-structure.github.io/conditional-composition/draft-vasters-json-structure-cond-composition.html
  JSTRUCT-ALTNAMES:
    title: "JSON Structure Alternate Names"
    author:
      - fullname: Clemens Vasters
    target: https://json-structure.github.io/alternate-names/draft-vasters-json-structure-alternate-names.html

--- abstract

This document defines a binding that allows an OpenAPI Specification (OAS)
{{OAS}} Schema Object to be expressed as a JSON Structure {{JSTRUCT-CORE}}
schema, using the schema-dialect selection mechanism — the `$schema` keyword
and the `jsonSchemaDialect` field, introduced in OAS 3.1 — that OAS already
provides. The binding is strictly additive and opt-in on a per-Schema-Object
basis. It introduces no new OpenAPI keywords, objects, or namespaces, and it
does not modify, fork, or republish the OpenAPI Specification.

--- middle

# Introduction {#introduction}

The OpenAPI Specification {{OAS}} (OAS) describes the data of requests,
responses, parameters, headers, and other message components with **Schema
Objects**. By default a Schema Object is a JSON Schema (Draft 2020-12) document
expressed in the OAS dialect.

Since OAS 3.1, the version that introduced Schema Object dialect selection,
OAS does not hard-wire that default. A Schema Object can declare a
different **dialect** through the `$schema` keyword, or inherit a
document-wide default through the OpenAPI Object's `jsonSchemaDialect` field.
This document uses that existing seam, and nothing else, to let a Schema
Object be a JSON Structure {{JSTRUCT-CORE}} schema.

The result is a strictly additive, opt-in binding:

* Adoption is per Schema Object. A single Schema Object, an entire
  `components.schemas` map, or a whole Description can use JSON Structure; the
  rest of the Description is unaffected.
* No new OpenAPI fields, objects, or namespaces are introduced. A JSON
  Structure Schema Object appears wherever a Schema Object is allowed and is
  referenced with an ordinary OpenAPI `$ref`.
* JSON Structure's precise type system, inheritance model, discriminated
  unions, cross-document reuse (`$import`), and unit annotations become
  available to API designers and to code generators.

This document defines how conforming tooling recognizes such a Schema Object,
how it is processed, and how its references resolve.

> **Note:** This document adds no fields to the OpenAPI Specification. It
> defines how JSON Structure is used within the Schema Object dialect
> mechanism that OAS has provided since version 3.1. OAS {{OAS}} remains
> authoritative for everything this document does not modify.

# Conventions and Terminology {#conventions-and-terminology}

{::boilerplate bcp14-tagged}

This document uses the following terms:

Schema Object:
: An OpenAPI Schema Object as defined by the OpenAPI Specification's Schema
  Object section {{OAS}}.

Dialect, meta-schema:
: The URI, carried in `$schema`, that identifies the language and version a
  schema resource is written in, as used by the OAS dialect-selection rules
  ("Specifying Schema Dialects") {{OAS}}.

JSON Structure Schema Object:
: A Schema Object whose effective dialect ({{declaring-a-json-structure-schema-object}})
  is one of the JSON Structure meta-schema URIs. Such a Schema Object is a
  JSON Structure schema and is processed per the JSON Structure
  specifications.

Schema resource:
: A self-contained JSON Structure schema identified by an `$id`, within which
  `$ref`, `$extends`, and `$import` are resolved.

Add-in:
: An optional JSON Structure keyword vocabulary (for example
  `JSONStructureValidation`, `JSONStructureUnits`) that a schema activates
  through `$uses`, subject to what its meta-schema offers.

Description:
: An OpenAPI Description (OAD) {{OAS}}: the complete set of documents
  describing an API.

# Scope and Relationship to the OpenAPI Specification {#scope}

This document is a profile, or binding, layered on OAS {{OAS}}, applicable to
any OpenAPI Description whose OAS version provides Schema Object dialect
selection through `$schema` and `jsonSchemaDialect` — introduced in OAS 3.1
and retained in every subsequent OAS 3.x version. It is normative only where
it states requirements about JSON Structure Schema Objects. In all other
respects, the OpenAPI Object, Paths, Operations, Media Types, Parameters,
Responses, Components, references between Objects, serialization, and
security schemes, OAS applies unchanged and is authoritative.

Where this document and OAS both describe the behavior of a Schema Object,
this document governs only those Schema Objects whose dialect is a JSON
Structure meta-schema ({{json-structure-meta-schema-uris}}). A Schema Object
that does not select a JSON Structure meta-schema is a plain OAS Schema
Object and is outside the scope of this document.

This document does not modify, fork, or republish the OpenAPI Specification.

# Declaring a JSON Structure Schema Object {#declaring-a-json-structure-schema-object}

## Dialect Selection {#dialect-selection}

A Schema Object is a JSON Structure Schema Object when its effective dialect
is a JSON Structure meta-schema URI. The effective dialect is determined
exactly as in the OAS dialect-selection rules (Section 4.24.7,
"Specifying Schema Dialects", of {{OAS}}):

1. If the Schema Object is a schema-resource root and carries `$schema`, that
   value is the dialect.
2. Otherwise, the OpenAPI Object's `jsonSchemaDialect` value, if present, is
   the dialect.
3. Otherwise, the OAS dialect applies and the Schema Object is NOT a JSON
   Structure Schema Object.

A `$schema` value on a resource-root Schema Object always overrides the
document default. Because JSON Structure schemas natively carry `$schema`, a
JSON Structure Schema Object is typically self-describing: pasting a JSON
Structure schema into a Schema Object position, with its `$schema` intact, is
sufficient.

## JSON Structure Meta-Schema URIs {#json-structure-meta-schema-uris}

The following URIs identify the JSON Structure dialects:

| Meta-schema URI | Meaning |
| ---- | ---- |
| `https://json-structure.org/meta/core/v0/#` | Core types and keywords only. |
| `https://json-structure.org/meta/extended/v0/#` | Core plus the `JSONStructureImport` add-in active by default, and offering (for opt-in through the schema's own `$uses`) the `JSONStructureAlternateNames`, `JSONStructureUnits`, `JSONStructureValidation`, and `JSONStructureConditionalComposition` add-ins. |
| `https://json-structure.org/meta/validation/v0/#` | The extended meta-schema with all add-ins, including validation and conditional composition, active by default. |

The extended meta-schema is RECOMMENDED as the document default for API
design, because it makes `$import` available while keeping validation and
composition opt-in.

These three URIs are the canonical JSON Structure dialects, but they are not
the only meta-schema URIs a Description may encounter. JSON Structure Core
permits a custom meta-schema to extend one of the canonical meta-schemas —
adding organization-specific keywords or add-ins — by giving the derived
meta-schema its own `$id` and using `$import` to pull in the foundational
meta-schema's definitions ({{JSTRUCT-CORE}}, "Meta-Schemas"). A Schema Object
whose `$schema` names such a derived meta-schema is a JSON Structure Schema
Object in every respect that matters to this binding, but its URI will not
appear in the table above and cannot be recognized by exact-match comparison
alone. {{tooling-requirements}} states how conforming tooling handles this
case.

## Tooling Requirements {#tooling-requirements}

* Tooling that supports this binding MUST recognize the three URIs in
  {{json-structure-meta-schema-uris}} and MUST process a Schema Object bearing
  one of them as a JSON Structure schema.
* Tooling that does not recognize a JSON Structure meta-schema URI MUST treat
  the Schema Object as an unknown dialect. It MUST NOT process the Schema
  Object as an OAS-dialect (JSON Schema Draft 2020-12) schema, because the
  keywords are interpreted differently ({{type-system}}).
* Matching against the URIs in {{json-structure-meta-schema-uris}} MUST be
  exact, byte-for-byte string comparison; tooling MUST NOT apply URI
  normalization, trailing-slash equivalence, or version-range matching. A
  future JSON Structure Core version that defines new meta-schema URIs is out
  of scope for this document and requires a new or updated binding.
* A `$schema` value naming a custom meta-schema that extends one of the three
  canonical meta-schemas ({{JSTRUCT-CORE}}, "Meta-Schemas") does not satisfy
  exact-match recognition and is out of scope for the MUST-recognize
  requirement above. Tooling MAY be separately configured to recognize
  specific custom meta-schema URIs — typically by resolving the meta-schema
  document once, offline, and confirming that it imports one of the three
  canonical meta-schemas — and, once so configured, MUST process Schema
  Objects bearing that URI as JSON Structure schemas under the canonical
  dialect it extends. Tooling that is not so configured MUST apply the
  unknown-dialect rule above rather than guess at the custom meta-schema's
  semantics.
* A resource-root JSON Structure Schema Object SHOULD carry `$schema`
  explicitly, even when the document default already selects a JSON Structure
  dialect, so that the schema remains self-describing when extracted from the
  Description.
* Tooling MUST construct and materialize default `$schema` and `$id` values
  per {{default-id-construction}} and {{materializing-defaults}} whenever a
  Schema Object lacking one or both keywords is extracted for standalone
  processing.

## Default `$id` Construction {#default-id-construction}

A resource-root JSON Structure Schema Object SHOULD carry its own `$id`. When
it does not, tooling still needs a stable, unique identifier for it — for
example, to run the Schema Object through a standalone JSON Structure
validator, or to resolve it as an `$import` target ({{cross-schema-reuse}}).
This section defines how tooling MUST construct that default, by combining
the Description's base URI with the Schema Object's location inside the
Description.

**Find the Description's base URI.** Tooling MUST try the following sources
in order, using the first one that applies. This is the same precedence OAS
itself uses for base URI determination (Appendix F, "Examples of Base URI
Determination and Reference Resolution", of {{OAS}}):

1. the OpenAPI Object's `$self` field, if present (OAS Appendix F.1);
2. otherwise, the base URI of an encapsulating entity, if any — for example,
   a `multipart/related` archive (OAS Appendix F.2);
3. otherwise, the Description's retrieval URI: the URI it was actually
   fetched from (OAS Appendix F.3);
4. otherwise, an application-specific default base URI, if the tooling
   defines one (OAS Appendix F.4).

If none of these applies — for example, the Description was authored offline
and never assigned a location or a `$self` value — tooling MUST require an
explicit `$id` on the Schema Object and MUST NOT proceed with extraction,
import resolution, or standalone validation until one is supplied.

**Find the Schema Object's location.** Tooling MUST determine the JSON
Pointer {{RFC6901}} from the Description's document root to the Schema
Object, for example `/components/schemas/TelemetryMessage`.

**Combine the two.** The default `$id` is the base URI with that JSON Pointer
appended as a fragment:

    <base-URI>#<JSON-Pointer>

For example, if the Description's base URI is
`https://example.com/api/openapi.yaml` and the Schema Object is at
`/components/schemas/TelemetryMessage`, the default `$id` is
`https://example.com/api/openapi.yaml#/components/schemas/TelemetryMessage`.

This mirrors how a JSON Schema resource embedded without its own `$id`
inherits identity from its position in the containing document, and it
guarantees a default that is stable and unique per Schema Object location,
independent of any `$ref`/`$import` graph resolution. An explicit `$id` on
the Schema Object always takes precedence over this default.

## Materializing Defaults for Standalone Validation {#materializing-defaults}

A JSON Structure Schema Object embedded in an OpenAPI Description commonly
omits `$schema` (relying on `jsonSchemaDialect`, {{dialect-selection}})
and/or `$id` (relying on {{default-id-construction}}). Both are
OAS-context-dependent conveniences: a standalone JSON Structure validator has
no notion of `jsonSchemaDialect` and no notion of the Schema Object's
position within a larger document.

Tooling that extracts a JSON Structure Schema Object from its containing
Description in order to validate it, import it, or otherwise process it
outside OAS context MUST first materialize both defaults as literal
properties on the extracted document:

* `$schema`, set to the effective dialect determined per
  {{dialect-selection}};
* `$id`, set to the Schema Object's own `$id` if present, or otherwise to the
  value constructed per {{default-id-construction}}.

The extracted document MUST be self-describing and independently valid JSON
Structure without reference to the originating Description. Tooling MUST NOT
hand an extracted Schema Object to a standalone JSON Structure validator with
`$schema` or `$id` left absent on the assumption that OAS-level context will
supply them.

A root JSON Structure schema that carries a `type` attribute also MUST carry
a `name` {{JSTRUCT-CORE}}, independent of the `$schema`/`$id` materialization
above and regardless of the root's type kind (not only `object`/`tuple`).
Within an OpenAPI Description, this requirement applies to every
`components.schemas` entry, since each is independently extractable and
validated as a schema resource root. An anonymous, inline Schema Object at a
request body, response, or parameter position becomes a schema resource root
the same way once it is extracted for standalone validation, and tooling
performing that extraction MUST likewise materialize a `name` for it, for
example by deriving one from the Schema Object's JSON Pointer location
({{default-id-construction}}) when the Schema Object does not already supply
one.

# Type System {#type-system}

Within a JSON Structure Schema Object, keywords are interpreted per JSON
Structure {{JSTRUCT-CORE}}, which defines the full type system: primitive and
precise numeric types, the `object`/`array`/`set`/`map`/`tuple`/`choice`
compound types and their member/item/value declarations, nullability via type
union, and inheritance via `abstract` and `$extends`. These JSON Structure
constructs replace the correspondingly named OAS / JSON Schema constructs,
which do not apply in this dialect.

## Inheritance {#inheritance}

Inheritance is expressed with `abstract` and `$extends` ({{JSTRUCT-CORE}}).
The OAS `discriminator` object combined with `allOf` is not an inheritance
mechanism in a JSON Structure Schema Object, and MUST NOT be used to express
type extension.

## Discriminated Unions {#discriminated-unions}

Discriminated unions use the JSON Structure `choice` type, in either its
tagged-union or inline-union form, with `selector` naming the discriminant
property ({{JSTRUCT-CORE}}).

A JSON Structure Schema Object MUST NOT include the OAS `discriminator`
field: its presence makes the Schema Object non-conforming with this binding.
Tooling MUST NOT interpret an OAS `discriminator` field as a discriminator
when present, and MUST use `selector` as JSON Structure's own discriminator
mechanism instead.

## Conditional Composition {#type-system-conditional-composition}

Where present, the `allOf`, `anyOf`, `oneOf`, `not`, and `if`/`then`/`else`
keywords of the `JSONStructureConditionalComposition` add-in {{JSTRUCT-COMPOSITION}}
are boolean validation combinators only. They do not merge or intersect type
definitions and MUST NOT be interpreted as structural composition. Structural
reuse is expressed exclusively with `$extends` ({{inheritance}}) and `$import`
({{cross-schema-reuse}}).

## Construct Mapping {#construct-mapping}

For readers migrating a Schema Object from the OAS dialect, the following
constructs change meaning:

| OAS / JSON Schema construct | JSON Structure equivalent |
| ---- | ---- |
| `type: string` + `format: date-time` / `uuid` / `binary` | dedicated types `datetime`, `uuid`, `binary` |
| `type: integer` + `format: int64` | `int64` (and the other precise numeric types) |
| `nullable: true` | type union including `"null"` |
| `discriminator` + `allOf` for subtyping | `abstract` + `$extends` |
| `oneOf` of variants + `discriminator`, variants unrelated | `choice` with a `choices` map (tagged union, {{discriminated-unions}}) |
| `oneOf` of variants + `discriminator`, variants share a base | `choice` with `$extends` and `selector` (inline union, {{discriminated-unions}}) |
| `allOf` / `anyOf` / `oneOf` as structural composition | not used for composition; conditional-composition add-in is validation-only |
| `additionalProperties` (boolean or schema) | `additionalProperties` (unchanged: boolean, or a schema constraining additional members) |
| `required` array | `required` (unchanged; MAY also be an array of arrays for mutually exclusive alternative required sets) |
| `enum` | `enum` (unchanged) |
| `const` | `const` (unchanged) |
| `readOnly` / `writeOnly` | not part of the JSON Structure type system; retained as OAS Schema Object annotations alongside the dialect |
| `deprecated` | not part of the JSON Structure type system; retained as an OAS Schema Object annotation |
| `$ref` to another `components.schemas` entry | `$import` / `$importdefs` ({{cross-schema-reuse}}) |

# Reference Model {#reference-model}

A JSON Structure Schema Object is a self-contained schema resource. Two
reference layers coexist and do not mix:

* The OpenAPI reference layer: an outer OpenAPI `$ref` that targets a Schema
  Object as a whole (for example `#/components/schemas/TelemetryMessage`).
  This resolves per OAS {{OAS}}, unchanged, at any position in the
  Description — including inside a Schema Object that has not itself
  adopted the JSON Structure dialect, such as an array wrapper's
  `items: { "$ref": "#/components/schemas/Pet" }`.
* The JSON Structure reference layer: `$ref`, `$extends`, and `$import` used
  *within* a Schema Object that has adopted a JSON Structure dialect. These
  resolve as defined by JSON Structure {{JSTRUCT-CORE}}: `$ref` and
  `$extends` address named types under the schema's own
  `#/definitions/...`; cross-document reuse instead uses the
  `JSONStructureImport` add-in {{JSTRUCT-IMPORT}} ({{cross-schema-reuse}}).
  An inner `$ref` MUST NOT be used to reach a `components.schemas` entry.

Because dialect selection is per schema-resource-root ({{dialect-selection}}),
these layers are scoped, not global: the JSON Structure reference layer's
rules apply only once a `$ref` node is already inside a JSON Structure
resource's own body. An ordinary OAS `$ref` pointing *at* a JSON Structure
resource from outside it — as in the array-wrapper example above — stays on
the OpenAPI reference layer and is untouched.

An OAS/JSON Schema `$ref` is a general-purpose node-reference mechanism: a
sibling property that can stand in for an entire Schema Object at any
position (`{ "$ref": "#/components/schemas/Foo" }`). JSON Structure has no
equivalent general-purpose mechanism, and this is a deliberate design
principle rather than an omission: its `$ref` exists solely to name the type
referenced by a `type` attribute, and there is no other keyword position
where "substitute this node with the schema over there" is meaningful.
Consequently `$ref` MUST only appear as the value of a `type` attribute
(`{ "type": { "$ref": "#/definitions/Foo" } }` {{JSTRUCT-CORE}}) and MUST NOT
be used as a sibling property standing in for a whole schema.

## `$ref` Names a Type, Not a Node {#reference-model-ref-placement}

This principle is easy to lose sight of when moving a `properties`, `items`,
or `choices` member from a plain OAS/JSON Schema Schema Object into a JSON
Structure Schema Object, because the OAS reference syntax parses without
complaint at the same position. Assume the array schema below has itself
adopted the JSON Structure dialect (its own `$schema`, or an inherited
`jsonSchemaDialect`, already selects it):

~~~json
{
  "type": "array",
  "items": { "$ref": "#/components/schemas/Pet" }
}
~~~

This is NOT valid JSON Structure. `$ref` here is a sibling of `items` rather
than the value of a `type` attribute, and it targets a `components.schemas`
entry, which JSON Structure's own `$ref` cannot reach in any case
({{cross-schema-reuse}}): `$ref` and `$extends` are document-local,
addressing only the schema's own `#/definitions`. A generic JSON/YAML parser
accepts the form above without complaint, so the violation only surfaces
when the Schema Object is validated with a JSON Structure-aware tool. The
principled form names a type for `items` to hold and points that type
reference at a locally defined type:

~~~json
{
  "type": "array",
  "items": { "type": { "$ref": "#/definitions/Pet" } },
  "definitions": {
    "Pet": {
      "type": "object",
      "name": "Pet",
      "properties": {
        "name": { "type": "string" }
      },
      "required": ["name"]
    }
  }
}
~~~

If `Pet` is also used verbatim elsewhere in the Description as a whole
Schema Object (for example, as another operation's response body), an
ordinary OAS `$ref` to `#/components/schemas/Pet` is used at that Schema
Object position instead of duplicating it there too; only positions nested
inside another Schema Object's `properties`, `items`, or `choices` require
the local `definitions` copy, because those positions cannot carry an OAS
`$ref` of their own.

# Cross-Schema Reuse with `$import` and `$importdefs` {#cross-schema-reuse}

Because `$ref` and `$extends` are document-local, a JSON Structure Schema
Object cannot reach a type defined in another `components.schemas` entry
through them. Reuse across entries, and across documents, uses the
`JSONStructureImport` add-in {{JSTRUCT-IMPORT}}, which is active by default
under the extended and validation meta-schemas and defines the `$import` and
`$importdefs` keywords, their copy-not-link semantics, definition shadowing,
import error handling, and cycle rejection. This section defines only how
those keywords resolve against an OpenAPI Description.

## Resolving Imports within a Description {#resolving-imports-within-a-description}

When the URI of `$import`/`$importdefs` matches the `$id` of a JSON Structure
Schema Object registered in the same OpenAPI Description, the import is
satisfied from that entry. Tooling MUST resolve the URI against the `$id`
values in the Description, using the same reference resolution as an
ordinary `$ref` as illustrated in the OAS base-URI-determination examples
{{OAS}}, before attempting any network retrieval. A JSON Structure Schema
Object that is imported this way therefore MUST declare an `$id`. Matching
compares the fragment-free absolute URI: tooling MUST strip any fragment
from the `$import`/`$importdefs` value before comparing it, byte-for-byte,
against each registered `$id` in the Description; `$id` values MUST be
unique within a Description, and tooling MUST reject a Description that
registers the same `$id` more than once.

## Resolving External Imports {#resolving-external-imports}

When the URI matches no `$id` registered in the Description, it denotes a
self-contained JSON Structure schema resource hosted elsewhere. Tooling
retrieves that document over a secure transport such as HTTPS, validates that
it is a JSON Structure schema, and copies its definitions into the designated
namespace exactly as for an in-Description import. The external document need
not itself be an OpenAPI Description, so a single canonical type library can
be shared across many APIs. Import errors, deduplication, and cycle rejection
follow {{JSTRUCT-IMPORT}} unchanged; an importing entry MUST additionally
activate, through its meta-schema or its own `$uses`, any add-ins used by the
definitions it imports.

# Activating Add-ins with `$uses` {#annotations-and-units}

Under the extended meta-schema, non-Core add-ins are offered but not active; a
schema activates them through `$uses`, an array of add-in tokens declared at
the schema root. Under the validation meta-schema, all add-ins are active by
default.

Following the convention established by the add-in specifications themselves
{{JSTRUCT-UNITS}} {{JSTRUCT-VALIDATION}} {{JSTRUCT-COMPOSITION}}, `$uses` is
placed at the root of the JSON Structure schema document to activate add-ins
offered by that document's own `$schema` meta-schema. Tooling conforming to
this binding MUST support `$uses` in that position.

For example, `JSONStructureUnits` {{JSTRUCT-UNITS}} is offered, but not
active, under the extended meta-schema; a schema that wants to annotate
numeric properties with `unit`, `ucumUnit`, and `symbol` MUST list it in
`$uses`:

~~~ yaml
components:
  schemas:
    Measurement:
      $schema: https://json-structure.org/meta/extended/v0/#
      $uses:
        - JSONStructureUnits
      $id: https://api.example.com/schemas/measurement
      name: Measurement
      type: object
      properties:
        temperature:
          type: double
          ucumUnit: "Cel"
          symbol: "°C"
~~~

Omitting `JSONStructureUnits` from `$uses` while still using `unit`,
`ucumUnit`, or `symbol` leaves those keywords unrecognized under the extended
meta-schema; tooling MUST treat them as unknown properties in that case,
rather than silently applying `JSONStructureUnits` semantics. Under the
validation meta-schema, `JSONStructureUnits` is active by default and `$uses`
is not required to enable it.

# Code Generation and Runtime Validation {#code-generation-and-runtime-validation}

Processing a JSON Structure Schema Object separates memory layout from
constraint checking:

Code generation:
: The generated in-memory type SHOULD be derived solely from the JSON
  Structure Core structural keywords: `type`, `name`, `properties`, `items`,
  `values`, `tuple`, `choices`, `selector`, `abstract`, `$extends`,
  `required`, `additionalProperties`, and the `precision`/`scale` and
  `maxLength` storage constraints. Constraint keywords contributed by the
  `JSONStructureValidation` {{JSTRUCT-VALIDATION}} and
  `JSONStructureConditionalComposition` {{JSTRUCT-COMPOSITION}} add-ins are
  runtime validation rules and SHOULD NOT change the generated type.

Runtime validation:
: Whether those add-ins are active is determined by the selected meta-schema:
  opt-in through `$uses` under the extended meta-schema, or active by default
  under the validation meta-schema. Conditional-composition keywords are
  evaluated as boolean value validation ({{type-system-conditional-composition}}),
  never as structural composition.

This separation lets a generator produce a deterministic memory layout from
Core keywords while a gateway or server enforces the full validation and
composition rules against the same schema.

# Examples {#examples}

## A Single JSON Structure Schema Object {#example-single-schema-object}

The following `components.schemas` entry is a JSON Structure Schema Object:
its `$schema` selects the JSON Structure Core dialect, so `type`,
`properties`, and the primitive type names are interpreted per JSON
Structure rather than per the OAS dialect. It is referenced from elsewhere in
the OpenAPI Description with an ordinary OpenAPI `$ref` to
`#/components/schemas/TelemetryMessage`.

~~~ yaml
components:
  schemas:
    TelemetryMessage:
      $schema: https://json-structure.org/meta/core/v0/#
      $id: https://api.example.com/schemas/telemetry-message
      name: TelemetryMessage
      type: object
      properties:
        messageId:
          type: uuid
        timestamp:
          type: datetime
        sensorId:
          type: string
        reading:
          type: double
~~~

## Using `jsonSchemaDialect` as the Document Default {#example-json-schema-dialect-default}

An OpenAPI Object MAY set `jsonSchemaDialect` once, so that Schema Objects
need not repeat `$schema` individually ({{dialect-selection}}). Here the same
`TelemetryMessage` schema omits both its own `$schema` and its own `$id`; the
document default supplies the dialect, and the default-construction rule
({{default-id-construction}}) supplies the identifier.

~~~ yaml
openapi: 3.1.0
jsonSchemaDialect: https://json-structure.org/meta/core/v0/#
info:
  title: Telemetry API
  version: "1.0.0"
components:
  schemas:
    TelemetryMessage:
      name: TelemetryMessage
      type: object
      properties:
        messageId:
          type: uuid
        timestamp:
          type: datetime
        sensorId:
          type: string
        reading:
          type: double
~~~

Assuming this Description is retrieved from
`https://api.example.com/openapi.yaml`, tooling that extracts
`TelemetryMessage` for standalone validation
({{materializing-defaults}}) MUST produce:

~~~ yaml
$schema: https://json-structure.org/meta/core/v0/#
$id: "https://api.example.com/openapi.yaml\
  #/components/schemas/TelemetryMessage"
name: TelemetryMessage
type: object
properties:
  messageId:
    type: uuid
  timestamp:
    type: datetime
  sensorId:
    type: string
  reading:
    type: double
~~~

A Schema Object that carries its own `$schema` and/or `$id` (as in
{{example-single-schema-object}}) overrides the corresponding default; the
two defaults compose independently per keyword, not as an all-or-nothing
pair, and are not mutually exclusive with each other.

## Cross-Schema Reuse with `$import` {#example-cross-schema-reuse}

Here `TelemetryMessage` imports the sibling `CommonTypes` library by its `$id`
into the `Common` namespace, extends the imported `BaseMessage`, and
references the imported `GeoPoint`. Both entries carry an `$id`, so the
import resolves within the Description without any network access.

~~~ yaml
components:
  schemas:
    CommonTypes:
      $schema: https://json-structure.org/meta/core/v0/#
      $id: https://api.example.com/schemas/common
      definitions:
        BaseMessage:
          name: BaseMessage
          abstract: true
          type: object
          properties:
            messageId: { type: uuid }
            timestamp: { type: datetime }
        GeoPoint:
          name: GeoPoint
          type: object
          properties:
            lat: { type: double }
            long: { type: double }
    TelemetryMessage:
      $schema: https://json-structure.org/meta/extended/v0/#
      $id: https://api.example.com/schemas/telemetry
      $root: "#/definitions/TelemetryMessage"
      definitions:
        Common:
          $importdefs: https://api.example.com/schemas/common
        TelemetryMessage:
          name: TelemetryMessage
          type: object
          $extends: "#/definitions/Common/BaseMessage"
          properties:
            sensorId: { type: string }
            reading: { type: double }
            location:
              type: { $ref: "#/definitions/Common/GeoPoint" }
~~~

## Importing an External Type Library {#example-external-import}

When the import URI matches no `$id` in the Description, it denotes an
external schema resource. Here `OrderEvent` imports an external `people.json`
type library by its absolute URI into the `People` namespace and uses the
imported `Person` type. The external document is a self-contained JSON
Structure schema and need not be an OpenAPI Description.

~~~ yaml
components:
  schemas:
    OrderEvent:
      $schema: https://json-structure.org/meta/extended/v0/#
      $id: https://api.example.com/schemas/order-event
      name: OrderEvent
      type: object
      properties:
        customer:
          type: { $ref: "#/definitions/People/Person" }
      definitions:
        People:
          $import: https://types.example.com/people.json
~~~

# Conformance {#conformance}

A conforming JSON Structure Schema Object is a Schema Object whose effective
dialect is one of the meta-schema URIs in {{json-structure-meta-schema-uris}}
and whose content is a valid JSON Structure schema for that meta-schema,
including any add-ins it activates through `$uses`.

Conforming tooling MUST:

1. Determine the effective dialect of every Schema Object per
   {{dialect-selection}}.
2. Process a Schema Object bearing a JSON Structure meta-schema URI as a JSON
   Structure schema, and never as an OAS-dialect schema
   ({{tooling-requirements}}).
3. Resolve `$ref`/`$extends`/`$import`/`$importdefs` within a JSON Structure
   Schema Object per JSON Structure {{JSTRUCT-CORE}} and
   {{cross-schema-reuse}}, resolving in-Description imports by `$id` before
   any network retrieval.
4. Keep the OpenAPI and JSON Structure reference layers separate
   ({{reference-model}}).

Conforming tooling SHOULD honor the code-generation and runtime-validation
separation of {{code-generation-and-runtime-validation}}, and MUST reject
cyclic imports.

A Description that contains only plain OAS Schema Objects is unaffected by
this document and remains a conforming OpenAPI Description.

# Security Considerations {#security-considerations}

Import retrieval:
: `$import`/`$importdefs` can reference external URIs. Tooling MUST resolve
  in-Description `$id` matches before any network access, MUST use a secure
  transport such as HTTPS for external retrieval, and MUST constrain
  automatic retrieval with an allow-list of permitted hosts or URI prefixes,
  connection and read timeouts, and response size limits to prevent
  server-side request forgery and resource exhaustion against internal or
  otherwise unintended targets. Tooling SHOULD cache retrieved schemas.

Cyclic and pathological imports:
: Cyclic import chains MUST be rejected. Deeply nested or fan-out imports
  SHOULD be bounded to prevent denial of service during schema assembly.

Dialect confusion:
: Because Core, JSON Schema, and JSON Structure share keyword spellings (for
  example `type`, `properties`, `$ref`) with differing semantics, tooling
  MUST NOT process a JSON Structure Schema Object as an OAS-dialect schema,
  or vice versa. Misidentifying the dialect can silently change validation
  outcomes.

Untrusted schemas:
: A JSON Structure schema retrieved from an external source is untrusted
  input. Tooling MUST validate it against its declared meta-schema before use
  and SHOULD apply the same input-handling precautions as for any other
  externally retrieved document.

Otherwise, the security considerations of OAS {{OAS}} apply unchanged.

# IANA Considerations {#iana-considerations}

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
