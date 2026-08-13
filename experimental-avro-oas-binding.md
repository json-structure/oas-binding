# An Experimental Avro Binding for OpenAPI Descriptions

**Status:** Experimental. Not a specification, not a submission, and not
endorsed by the Apache Software Foundation or the Apache Avro project.

**Purpose:** To test whether Part I of
[draft-vasters-json-structure-oas-binding](draft-vasters-json-structure-oas-binding.md)
is actually dialect-neutral, by binding a second dialect that shares almost
nothing with the first.

**Method:** Supply the seven declarations of "Binding Parameters" and nothing
else. Where a rule of Part I already covers Avro, this document does not
restate it. Where Avro forces something Part I did not anticipate, this
document says so instead of quietly patching around it.

**Baseline:** Apache Avro Specification 1.12.0, OAS 3.2.0.

---

## Why Avro

A binding framework built while binding one dialect tends to be a
description of that dialect wearing a costume. JSON Structure is a close
relative of JSON Schema. It has a `$id`, it has document-local `$ref`, it
has a cross-document import mechanism, it has an identifier grammar, and its
schemas are JSON objects. Every one of the seven binding parameters has an
obvious value.

Avro has almost none of that:

- No `$schema`, and no dialect identifier of any kind.
- No `$id`, and no URI-based identity for a schema.
- No cross-document reference mechanism in the schema declaration format.
- Names are scoped by an author-declared `namespace`, not by the document.
- A schema is not necessarily a JSON object. It can be a JSON array (a
  union) or a JSON string (a named-type reference).
- A JSON encoding that is defined by the dialect and does not match the
  obvious one.

If the abstraction survives Avro, it is an abstraction. If it only survives
by acquiring Avro-shaped exceptions, it was a description of JSON Structure.

## Binding Parameters

| Parameter | Value for Avro |
| ---- | ---- |
| Dialect URIs | One placeholder URI, `https://example.org/dialects/avro/1.12#`. No vocabularies or add-ins. Logical types are part of the dialect and are always active. |
| Identity keyword | **None.** A resource is addressable only by location. |
| Identity comparison | Not applicable. |
| Cross-document mechanism | **None.** A schema resource MUST be self-contained. |
| Type determination | The procedure in [Type Determination](#type-determination-for-non-json-serializations). |
| Type naming and scope | A named type is named by its fullname. A resource's scope is the set of namespaces its declarations name. See [Type Naming and Scope](#type-naming-and-scope). |
| Self-description | Yes. An extracted resource MUST carry an explicit `namespace` on every named declaration that inherited one. |

Two of the seven go inert by declaration. Two are trivial. Three need real
text. That distribution is roughly what a parameterized framework should
produce.

### On the dialect URI

Avro has no meta-schema and no registered dialect URI, because it has no
`$schema` keyword to put one in. The URI above is a placeholder. A real
binding would need a URI minted by whoever owns the dialect. Nobody has
minted one, and this document does not propose that anyone should.

Avro permits attributes it does not define, as metadata that must not affect
the serialized form. A `$schema` attribute at the root of an Avro record
schema is therefore legal Avro and inert to Avro tooling. An author who
wants a self-selecting Avro Schema Object can write one. This works only for
the object form of an Avro schema, which is the next problem.

## What a Schema Object May Hold

An OAS Schema Object is a JSON object, or in the OAS dialect a boolean. It
is never a JSON array and never a JSON string.

Three of Avro's schema forms are therefore unusable at a Schema Object
position:

| Avro form | Example | Usable |
| ---- | ---- | ---- |
| Object | `{"type": "record", "name": "Pet", ...}` | yes |
| Object, primitive | `{"type": "string"}` | yes |
| String, primitive shorthand | `"string"` | no |
| String, named-type reference | `"com.example.Pet"` | no |
| Array, union | `["null", "string"]` | no |

A conforming Avro Schema Object MUST be a JSON object. An author whose
schema is a union at the top level MUST wrap it, either in a record with a
single field or in a `{"type": ...}` object form.

Inside a Schema Object, the excluded forms are unrestricted. `"string"` and
`["null", "string"]` and `"com.example.Pet"` are all ordinary field types.
The restriction applies to the resource root and nowhere else, which is the
same shape as the resource-root rules of Part I.

Part I does not state this constraint. See
[What Did Not Hold](#what-did-not-hold).

## Identity

Avro has no identity keyword. This binding declares none, and "Default
Resource Identity" therefore does not apply. Avro schema resources in an
OpenAPI Description are addressable by their OAS location and by nothing
else.

Nothing downstream misses it. "Resolving Cross-Document References" is
already inert, because there is no cross-document mechanism to resolve.
"Materializing Defaults for Standalone Processing" reduces to `$schema` plus
whatever the self-description parameter adds, and the identity bullet is
vacuous.

The Part I escape hatch is written as an aside. In Avro it is load-bearing.

## Cross-Schema Reuse

There is none.

Avro's named types are addressable by fullname within one schema, subject to
the define-before-use rule. That rule is scoped to a single schema. The
schema declaration format has no mechanism to pull a definition in from
another document. Avro IDL has an `import`, and schema registries invent
their own, but neither is part of the schema declaration format that occupies
a Schema Object position.

Consequently:

- An Avro Schema Object MUST define every named type it references.
- Two `components.schemas` entries that both need `com.example.Pet` MUST each
  define it.
- The only reuse available is an OAS `$ref` at the OpenAPI layer, which
  targets a whole Schema Object.

This is the same trap as the JSON Structure binding's `$ref` placement rule,
arrived at from a different direction. A field type inside an Avro record
cannot carry an OAS `$ref`, so a type needed at a nested position must be
copied. Part I's "Reference Layer Separation" covers this without amendment.

## Self-Description: Materializing `namespace`

This is where Avro earns its place in the experiment.

Avro determines a named type's fullname in three ways. If `name` contains a
dot, it is the fullname. If `name` and `namespace` are both given, the
fullname is their concatenation. If only a dotless `name` is given, **the
namespace is taken from the most tightly enclosing named schema**, and the
null namespace applies if there is none.

The third case makes a nested declaration's identity depend on its
container. Extract it, and its fullname silently changes.

~~~json
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

`LineItem` here is `com.example.sales.LineItem`. Lift that inner record out
for standalone processing and it becomes `LineItem` in the null namespace. It
is a different type with the same JSON text.

Tooling that extracts an Avro Schema Object, or any declaration nested within
one, MUST first write the inherited `namespace` explicitly onto every named
declaration that inherited it. The materialized value MUST be the namespace
in effect at that declaration's position in the original document.

That is the self-description parameter doing exactly what it is for. But the
prose that consumes it does not quite fit. See
[What Did Not Hold](#what-did-not-hold).

## Type Naming and Scope

A named type is named by its fullname. Anonymous forms (`array`, `map`,
`union`, and the primitives) name nothing and are out of scope for the
aggregation rule.

A resource's scope is the set of namespaces its declarations name. It is not
one namespace, and it is not derived from the resource. An author picks the
namespace per declaration, and a single Avro schema can populate several.

Where two schema resources of one Description declare the same fullname with
identical definitions, the aggregate holds one declaration. That is not a
merge. It is Avro's own rule that a fullname has one definition.

Where two schema resources declare the same fullname with differing
definitions, Avro cannot express the result. Tooling MUST report a
diagnostic. It MUST NOT rename either declaration, MUST NOT pick one, and
MUST NOT synthesize a disambiguating namespace, because a synthesized
namespace changes the fullname that consumers of the original resource
already depend on.

This exercises the branch of Part I's rule that JSON Structure never reaches.
JSON Structure can always express the resulting scopes, because its
namespaces are constructed from resource names. Avro cannot, because its
namespaces are declared by the author. The diagnostic branch is not
defensive drafting. It is the normal outcome for a dialect whose naming is
author-controlled.

## Type Determination for Non-JSON Serializations

Avro defines its own JSON encoding. That encoding, and not intuition about
the type names, governs.

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

Tooling MUST determine the type of a value as follows. All steps operate on
the schema alone.

1. Resolve the starting point. Follow a named-type reference to its
   definition within the same schema. Resolution is by fullname, with the
   namespace-inheritance rule applied.
2. Locate the value: a field under `fields`, the item type under `items`, or
   the value type under `values`.
3. Take the located declaration's Avro type and map it through the table
   above.
4. Ignore `logicalType`. A logical type annotates an underlying Avro type and
   does not change the JSON encoding. `{"type": "long", "logicalType":
   "timestamp-millis"}` is a **number**. `{"type": "int", "logicalType":
   "date"}` is a **number**. `{"type": "bytes", "logicalType": "decimal"}` is
   a **string**.
5. A union yields "undetermined".

Two of these will surprise anyone arriving from JSON Schema or JSON
Structure.

**Dates and timestamps are numbers.** Avro encodes `date` as days since the
epoch and `timestamp-millis` as milliseconds since the epoch, both as JSON
numbers. Tooling MUST NOT serialize them as formatted date strings in a form
field, path segment, query parameter, or header value.

**A nullable value is not its non-null branch.** Avro's JSON encoding tags
union values: `["null", "string"]` encodes as `null` or as
`{"string": "..."}`, never as a bare string. Tooling MUST NOT collapse a
two-branch union with a `null` branch to the other branch's type. The JSON
Structure binding permits exactly that collapse, and it is correct there,
because JSON Structure's type unions are untagged. Carrying the shortcut
across dialects produces a wrong wire type with no error raised.

That is Part I's "dialect confusion" hazard in its most concrete form. The
two dialects have the same feature, spelled almost the same way, with
different wire consequences.

## Worked Example

An OpenAPI Description with two Avro Schema Objects, one of which needs the
other's type at a nested position and therefore defines its own copy.

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
        - name: name
          type: string
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
                - name: name
                  type: string
                - name: tags
                  type: { type: array, items: string }
~~~

Both entries validate as Avro on their own. Aggregated, they collide: two
definitions of `com.example.petstore.Pet` with different fields. The inner
one inherited its namespace from `PetListResponse`.

Under the scope rule above, tooling reports a diagnostic and stops. It does
not rename the inner record, and it does not push it into a synthesized
namespace, because either would change a fullname that an Avro consumer of
`PetListResponse` already reads off the wire.

The author fixes it in the Description, by giving the inner record its own
namespace or by making the two definitions identical. That is the same
outcome the JSON Structure binding reaches, by a mechanism the JSON Structure
binding does not have.

## Scorecard

### Applied unchanged

No Avro-specific text was needed for any of these:

- Reference Object Classification
- Dialect Selection
- Recognizing and Rejecting Dialects
- Reference Layer Separation
- Validating the Description Itself
- Conformance roles
- Security: dialect confusion, untrusted schemas

### Inert by declaration

The escape hatches were used, and nothing downstream broke:

- Default Resource Identity, and with it the whole JSON Pointer construction
- Resolving Cross-Document References
- Security: cross-document retrieval, resources outside the OAD, cyclic and
  pathological import graphs

### Required real work, as designed

- Type determination
- Self-description
- Type naming and scope

## What Did Not Hold

Three findings. All three are defects in Part I, not in Avro.

**1. "Resource boundary as scope boundary" is wrong for author-scoped
dialects.**

Part I says tooling "MUST preserve the resource boundary as a scope boundary"
and "MUST NOT merge the type declarations of two schema resources into a
single scope."

Avro cannot satisfy that while remaining Avro. Two resources that both
declare types in `com.example.petstore` are in one scope because the author
put them there. Refusing to merge them would contradict the Description.

The rule generalizes badly from JSON Structure, where scope is derived from
the resource, to any dialect where scope is declared inside the schema.

Proposed repair: state the invariant instead of the mechanism. Tooling MUST
NOT introduce a collision that does not exist in the Description, and MUST
NOT resolve a collision by renaming. Whether that requires preserving the
resource boundary depends on the dialect, and the binding says which.

**2. The self-description parameter is broader than the section that
consumes it.**

"Materializing Defaults for Standalone Processing" frames materialization as
recovering "conveniences that depend on OAS context" — the dialect from
`jsonSchemaDialect`, the identity from the Schema Object's position.

Avro's inherited `namespace` is neither. It is a default supplied by the
*dialect's own* containment rule, which the extraction breaks just as
thoroughly. The parameter slot in the table accommodates it. The prose that
sends tooling to that slot does not describe it.

Proposed repair: extend the section to cover any value the extracted document
would otherwise inherit from context, whether that context is the OpenAPI
Description or the dialect's own nesting.

**3. Part I never says a Schema Object must be a JSON object.**

It is inherited from OAS and true throughout, so it never came up while
binding a dialect whose schemas are always objects. It is the first thing
that bites a dialect whose schema *document* form is sometimes an array or a
string, and Avro is not unusual in that respect.

Proposed repair: one sentence in "Dialect Selection". A dialect whose schema
forms include non-object JSON values is bound only for its object forms, and
the binding says which forms are usable at a Schema Object position.

## Conclusion

The framework held for four of the seven parameters without amendment,
degraded cleanly for the two Avro does not have, and produced usable
requirements for the one it does have differently.

The three failures are all in Part I's *prose*, not its structure. Each is a
place where a rule was written against the dialect at hand rather than
against the parameter it belongs to. That is the specific failure mode a
second binding is supposed to catch, and it caught three.

Binding Avro took one document and no new machinery. That is the result the
experiment was looking for.
