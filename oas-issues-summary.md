# Gaps in OAS 3.2 Dialect Support, and Proposals to Close Them

**Status:** input for discussion with the OpenAPI Initiative.
**Target:** OAS 3.3, or 4.0 where a change is not backward compatible.
**Baseline:** OpenAPI Specification 3.2.0, published 19 September 2025.

## Why this document exists

`draft-vasters-json-structure-oas-binding` binds JSON Structure into OpenAPI
Descriptions through the `$schema` / `jsonSchemaDialect` mechanism. Writing it
turned up an uncomfortable ratio: roughly two thirds of its normative
requirements have nothing to do with JSON Structure. They are answers to
questions any dialect binding must answer, and OAS does not answer them.

The binding therefore has two parts. Part I, "Dialect Binding Requirements,"
is dialect-agnostic and parameterized by a handful of declarations a binding
supplies. Part II binds JSON Structure to it. Part I exists only because OAS
defines dialect *selection* and stops there.

**The goal of this document is to make Part I unnecessary.** Each section
below states what OAS says today, what it leaves undefined, why that matters,
and a concrete proposal. The final section maps each proposal to the text it
would let us delete.

## The underlying problem

OAS 3.1 shipped `jsonSchemaDialect` and, with it, an extension point.
[Section 4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects)
says a Schema Object may declare a dialect, that tooling MUST support
the OAS dialect, and that it MAY support others. Everything downstream of that
sentence is undefined: what a tool does when it does not recognize the URI,
whether URI matching is exact, what identity an embedded schema resource has
when it declares none, how the OpenAPI reference layer relates to the
dialect's own, how a non-JSON body is parsed when the schema is not JSON
Schema, and what "conforming tooling" means when tools do disjoint jobs.

That was survivable while the OAS dialect was the only dialect in practice.
With one dialect there is nothing to disambiguate: the two reference layers
blur together harmlessly because both are JSON Schema; the identity question
does not bite because nobody extracts schemas; the unknown-dialect question
never arises.

It bites the moment a second dialect exists. And if the next binding answers
these questions independently, it will answer them differently — at which
point a tool supporting two dialects has two contradictory conformance
obligations for the same code path. Dialect bindings are where per-spec
divergence costs most, because the point of a binding is that one reader
handles all of them.

## OAS already admits mutually incompatible dialects

OAS specifies how to select a non-default dialect.
[§4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects)
says `$schema`
"MAY be present in any Schema Object that is a schema resource root, and if
present MUST be used to determine which dialect should be used when
processing the schema," that this "allows use of Schema Objects which comply
with **other drafts of JSON Schema** than the default Draft 2020-12 support,"
and that tooling "MAY support additional values of `$schema`."
`jsonSchemaDialect` extends the same choice to a whole document. An OAS
editor confirmed the intended usage in September 2025 (#4147): a Description
may set `jsonSchemaDialect: http://json-schema.org/draft-07/schema#`.

Those drafts are not variations on a theme. JSON Schema's own release notes
document each transition as breaking, and several of the breaks are silent —
the same document validates differently under the new draft, with no error:

| Transition | Change (per JSON Schema release notes) | Effect on an unmodified document |
|---|---|---|
| draft-04 → 06 | `id` renamed `$id` | The base URI declaration stops being recognized |
| draft-04 → 06 | `exclusiveMinimum`/`exclusiveMaximum` change from boolean to number | Schema becomes meta-schema-invalid, or the bound is silently dropped |
| draft-04 → 06 | `"integer"` redefined as "any number with a zero fractional part" | `1.0` is invalid before, valid after — **silent** |
| draft-07 → 2019-09 | `$ref` changes from replacing the schema to an applicator whose siblings apply | Adjacent keywords go from ignored to enforced — **silent** |
| draft-07 → 2019-09 | `format` stops being an assertion by default | Constraints stop being enforced — **silent** |
| draft-07 → 2019-09 | `definitions` → `$defs`; `dependencies` split into `dependentSchemas`/`dependentRequired`; `$id` may no longer carry a fragment | Reference targets and constraints stop being recognized |
| 2019-09 → 2020-12 | Array-form `items` becomes `prefixItems`; `items` takes over `additionalItems` | "The meaning of `items` has changed, the syntax for defining arrays remains the same" — **silent** |
| 2019-09 → 2020-12 | `contains` now marks items evaluated for `unevaluatedItems` | The spec's own example flips from fail to pass — **silent** |
| 2019-09 → 2020-12 | `$recursiveRef`/`$recursiveAnchor` replaced by `$dynamicRef`/`$dynamicAnchor` | Recursive references stop resolving |

Nothing commits JSON Schema to stopping. `$schema` exists because drafts are
not interchangeable, and the 2020-12 notes are explicit about the
consequence: "Implementations need to be prepared to switch processing modes
or throw an error if they don't support the `$schema` of the referenced
schema." JSON Schema mandates the no-fallback behavior that OAS leaves
undefined (gap 3 below).

The gaps in this document are therefore not hypothetical, and not specific to
JSON Structure. They already produce undefined behavior for the drafts OAS
already sanctions:

- [Appendix F](https://spec.openapis.org/oas/v3.2.0.html#appendix-f-examples-of-base-uri-determination-and-reference-resolution)
  and [§4.1.2.1](https://spec.openapis.org/oas/v3.2.0.html#parsing-documents)
  name `$id` as the Schema Object's identity keyword.
  A draft-04 Schema Object declares `id`. OAS's base-URI machinery does not
  apply to a dialect OAS permits. (Gap 5.)
- A tool that has not implemented draft-07 has no defined behavior on
  encountering it. (Gap 3.)
- Nothing says whether `http://json-schema.org/draft-07/schema#` and
  `https://json-schema.org/draft-07/schema` select the same dialect. JSON
  Schema's own URIs vary in scheme and trailing `#` across drafts. (Gap 4.)
- `schema-base` pins `jsonSchemaDialect` to the OAS dialect, so a draft-07
  Description already fails it. (Gap 7.)

JSON Structure sits on the same axis: another dialect, selected by the same
keyword, assigning its own meanings to shared keyword spellings. Admitting it
introduces no category of risk that admitting draft-07 did not. The proposals
below are not accommodations for JSON Structure. They are what the existing
extension point needs in order to work for the dialects OAS already
sanctions.

---

## 1. Dialect processing is undefined

**Today.**
[Section 4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects)
defines selection.
[Section 4.24](https://spec.openapis.org/oas/v3.2.0.html#schema-object)
defines the Schema
Object as "a superset of the JSON Schema Specification Draft 2020-12."
Nothing defines what it means to *process* a Schema Object whose dialect is
not JSON Schema.

**Gap.** There is no concept of a "dialect binding" — a specification that
makes a dialect usable in a Description — and therefore no statement of what
such a specification must supply.

**Proposal.** Add a section, "Schema Object Dialect Bindings," defining the
term and requiring a binding to declare:

| Declaration | Purpose |
|---|---|
| Dialect URIs | Which exact URIs select the dialect, and per URI, which vocabularies are active by default versus offered for opt-in. |
| Identity keyword | Which keyword carries a schema resource's identity (JSON Schema: `$id`), or that the dialect has none. |
| Identity comparison | Byte-exact, or a declared normalization. |
| Cross-document mechanism | Which keyword(s), if any, incorporate definitions from another resource, and whether their values must be absolute URIs. |
| Type determination | The procedure yielding a value's JSON data type from the schema, for non-JSON serializations (see gap 6 below). |
| Self-description | Which keywords must be materialized when a resource is extracted from the Description. |

Everything else in this document is expressible in terms of those six
declarations. The OAS dialect itself satisfies them trivially and can be
described as the built-in binding — a check that the parameter set
generalizes beyond JSON Structure.

## 2. Reference Object versus Schema Object classification is not ordered

**Today.**
[Section 4.23](https://spec.openapis.org/oas/v3.2.0.html#reference-object)
defines the Reference Object with fixed fields `$ref`,
`summary`, `description`, and states it "cannot be extended with additional
properties, and any properties added SHALL be ignored," noting that this is a
difference from a Schema Object containing `$ref`.

**Gap.** OAS never says *when* the classification decision is made relative to
dialect selection. With a single dialect this does not matter, because a
Reference Object and a Schema Object containing `$ref` are both processed by
the same JSON Schema machinery. With a second dialect it determines which
processor sees the object: a bare `{ "$ref": "..." }` under
`components.schemas` would otherwise inherit the document `jsonSchemaDialect`
and be handed to a dialect processor that has no idea what to do with it.

**Proposal.** Add to
[§4.23](https://spec.openapis.org/oas/v3.2.0.html#reference-object) or
[§4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects):
tooling MUST classify a JSON object at
a Schema Object position as a Reference Object or a Schema Object before
performing dialect selection, MUST classify by the fixed-field test alone, and
MUST re-classify at each position independently. Dialect selection applies
only to Schema Objects. An object that carries `$ref` alongside any property
outside the fixed set is not a valid Reference Object and MUST be rejected as
invalid at the OAS layer; tooling MUST NOT reinterpret the extra properties as
dialect Schema Object content.

That last rule requires OAS to pick a reading it currently leaves open. "Any
properties added SHALL be ignored" and "cannot be extended" pull in opposite
directions: ignoring extra properties implies the object is still a Reference
Object, while "cannot be extended" implies it is not one. A third option —
reclassifying it as a Schema Object — is worse than either, because it hands a
dialect processor an object the author plainly wrote as a reference. Rejection
is the reading this binding assumes, and OAS should state it explicitly.

## 3. Unknown dialects have no defined behavior

**Today.** "Tooling MUST support the OAS dialect schema id, and MAY support
additional values of `$schema`"
([§4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects)).

**Gap.** MAY support says nothing about what happens when a tool does not.
The dangerous default — and the one implementations reach for — is to parse
the object as an OAS-dialect schema anyway, because it is a JSON object with
plausible-looking keywords. `type`, `properties`, and `$ref` exist in many
dialects with materially different meaning. Silent misinterpretation changes
validation outcomes without any error.

**Proposal.** Add a no-fallback rule to
[§4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects):

> Tooling that does not recognize the effective dialect of a Schema Object
> MUST treat that Schema Object as being of an unknown dialect. It MUST NOT
> process the Schema Object as an OAS-dialect schema. A reader MUST preserve
> the Schema Object and its containing Description for pass-through or
> diagnostic handling and MUST NOT deserialize it as an OAS Schema Object.
> Tooling MAY fail the operation when it cannot preserve or expose the
> object, but MUST report the unknown dialect as the cause.

This is the change worth making first. It is small and backward compatible,
and without it a multi-dialect ecosystem fails silently instead of loudly.

## 4. Dialect URI matching semantics are unspecified

**Today.** Nothing states how a `$schema` or `jsonSchemaDialect` value is
compared against a dialect a tool supports.

**Gap.** URI comparison admits several defensible answers — byte-exact,
RFC 3986 normalized, trailing-slash-insensitive, version-range. Different
tools will pick differently, and a dialect URI is an identity, not a location,
so a near-miss must not silently succeed.

**Proposal.** State in
[§4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects)
that matching is exact, byte-for-byte string
comparison unless the dialect's own specification defines a normalization;
that tooling MUST NOT apply URI normalization, trailing-slash equivalence, or
version-range matching on its own initiative; and that a dialect version
defining new URIs is a distinct dialect for matching purposes.

## 5. An embedded schema resource without an identity has no defined identity

**Today.**
[Appendix F](https://spec.openapis.org/oas/v3.2.0.html#appendix-f-examples-of-base-uri-determination-and-reference-resolution)
defines base URI determination in four ordered steps:
content (`$self` for OpenAPI documents, `$id` for Schema Objects), then
encapsulating entity, then retrieval URI, then an application-specific
default.
[Section 4.1.2.1](https://spec.openapis.org/oas/v3.2.0.html#parsing-documents)
lists `$self`, `$id`, `$anchor`, and `$dynamicAnchor`
as reference targets.

**Gap.** Two related holes. First, when a `components.schemas` entry declares
no `$id`, it has a base URI but no identity, so nothing can name it except its
location within the entry document. Second, OAS says nothing about
*extraction* — lifting a Schema Object out of a Description to hand it to a
standalone validator for its dialect. Extraction drops exactly the context
(`jsonSchemaDialect`, position) that made the omissions safe.

**Proposal.** Add a subsection, "Identity of Embedded Schema Resources":

- Define a default identity for a Schema Object that declares none: the
  Description's base URI per
  [Appendix F](https://spec.openapis.org/oas/v3.2.0.html#appendix-f-examples-of-base-uri-determination-and-reference-resolution),
  with the Schema Object's JSON Pointer
  as an RFC 3986 fragment. Specify the construction deterministically —
  UTF-8 basis, RFC 6901 token escaping applied before percent-encoding,
  literal `%` encoded as `%25`, uppercase hex digits — because the result
  participates in identity comparison.
- State that this default is for standalone processing only and is not a
  target for any dialect's cross-document mechanism; only an explicitly
  declared identity is.
- Require that tooling extracting a Schema Object for standalone processing
  first materialize `$schema` (the effective dialect) and the identity keyword
  as literal properties, plus any further keywords the binding declares.

## 6. Type determination for non-JSON serializations is dialect-locked

**Today.**
[Section 4.24.4.2](https://spec.openapis.org/oas/v3.2.0.html#non-json-data)
requires implementations to inspect the schema to
determine a value's type before parsing or serializing non-JSON content, and
specifies the procedure concretely: examine the starting schema and every
schema reachable "by following only `$ref` and `allOf` keywords," with stated
handling for ambiguous `type` lists and for `anyOf`/`$dynamicRef`.

**Gap.** That procedure is written entirely in OAS-dialect keywords. A Schema
Object in another dialect at a `application/x-www-form-urlencoded`,
`multipart`, `text/plain`, Parameter, Header, or Encoding position has no
defined type-determination procedure at all. Alone in this list, this gap
breaks function rather than tidiness: without a procedure, the data cannot be
parsed correctly. The 3.2 additions — `itemSchema`, `prefixEncoding`,
`itemEncoding`, sequential media types — all inherit it.

**Proposal.** Refactor
[§4.24.4.2](https://spec.openapis.org/oas/v3.2.0.html#non-json-data)
into two layers:

1. A dialect-neutral contract: given a starting-point Schema Object and a
   value locator (property name, array position, item), the procedure returns
   a JSON data type or "undetermined"; it MUST be deterministic and MUST NOT
   depend on instance data; "undetermined" MUST produce a diagnostic rather
   than a guess.
2. The existing `$ref`/`allOf` walk, restated as the OAS dialect's
   implementation of that contract.

Then require each binding to supply its own implementation, and state that a
dialect whose binding supplies none MUST NOT be used at the affected
positions.

## 7. No published schema validates a Description that uses another dialect

**Today.** Two JSON Schemas are published per OAS minor version. `schema`
deliberately does not validate Schema Objects, "as they make no assumptions
about the JSON Schema dialect in use." `schema-base` does validate them, and
pins `jsonSchemaDialect` and `$schema` to the OAS dialect with `const`.

**Gap.** A Description carrying non-OAS-dialect Schema Objects validates
against `schema` and fails against `schema-base`. There is no published schema
that performs full structural validation of everything *except* the Schema
Objects' internals while permitting a different dialect — which is what a
multi-dialect Description needs, and what OAS's own default behavior already
does.

This is already recognized: OAI issue #4147 asks for "an intuitively-named
schema that performs the default accepted Schema Object validation but still
allows switching metaschemas," and observes that `schema-base` is "overly
restrictive."

**Proposal.** Publish a third iteration per minor version — the naming is the
open question in #4147, not the need — that validates all OAS Objects fully
and treats a Schema Object as `type: [boolean, object]` when its effective
dialect is not the OAS dialect. Until then, document in the specification text
that `schema-base` failures arising solely from a non-OAS dialect URI are
expected and are not defects.

## 8. The status of non-OpenAPI, non-JSON-Schema documents is undefined

**Today.**
[Section 4.1.2](https://spec.openapis.org/oas/v3.2.0.html#openapi-description-structure)
requires that "all documents in an OAD MUST have
either an OpenAPI Object or a Schema Object at the root," and that documents
with a different root "MAY be supported, but will have implementation-defined
or, potentially, undefined behavior."
[Appendix G](https://spec.openapis.org/oas/v3.2.0.html#appendix-g-parsing-and-resolution-guidance)
lists detection mechanisms,
all of which assume OpenAPI or JSON Schema.

**Gap.** A dialect with a cross-document mechanism will reach documents that
are neither. A shared JSON Structure type library pulled in by `$import` is a
JSON Structure schema document, not an OpenAPI Object and not a JSON Schema.
Under the current text it lands in the undefined-behavior bucket — a poor
answer for a mechanism the dialect defines and OAS invited.

**Proposal.** State that a resource obtained through a dialect's own
cross-document mechanism is **not** a document of the OAD. It is an input to
dialect processing, governed by the dialect's rules; OAS document-structure
requirements do not apply to it; tooling MUST NOT parse it as an OpenAPI
document and MUST NOT expose its contents at OAS component addresses. This
costs OAS nothing and removes an unnecessary conformance cliff.

## 9. There is no conformance role model

**Today.** OAS uses "tooling" throughout without distinguishing what a given
tool does. There is no conformance section.

**Gap.** A documentation renderer, a request validator, a code generator, and
a bundler have almost disjoint obligations, and a requirement addressed to
"tooling" is either over-broad or unenforceable. This is a large part of why
"is this tool OAS-compliant?" has no answer today, and it gets worse with
multiple dialects, where a tool may legitimately read a dialect it cannot
generate code for.

**Proposal.** Add a Conformance section defining roles — Reader, Schema
Validator, Resolver, Instance Validator, Codec / Code Generator are the set
the binding found necessary — each with its own requirements, and state that
an implementation may conform to any subset. Add the rule that a tool
implementing several roles MUST keep their failure modes distinct: a validator
rejection must not be silently downgraded by a generator operating on the same
schema.

This has the broadest benefit outside the dialect question, and the largest
editorial cost.

## 10. External retrieval security requirements are too thin to implement

**Today.**
[Section 6.5](https://spec.openapis.org/oas/v3.2.0.html#handling-external-resources),
"Handling External Resources," is two sentences noting
that external resources may be untrusted.
[Section 6.6](https://spec.openapis.org/oas/v3.2.0.html#handling-reference-cycles),
"Handling Reference
Cycles," is two sentences requiring that cycles be detected.
[Section 4.1.2.2.1](https://spec.openapis.org/oas/v3.2.0.html#establishing-the-base-uri)
notes that implementations MAY support retrieval and points at the security
considerations.

**Gap.** No ordering (when is the network consulted relative to in-document
resolution?), no default (is retrieval on or off?), no limits, and cycle
detection is stated without saying when it must happen relative to expansion.
Every binding will otherwise reinvent these, differently.

**Proposal.** Strengthen
[§6.5](https://spec.openapis.org/oas/v3.2.0.html#handling-external-resources)
and
[§6.6](https://spec.openapis.org/oas/v3.2.0.html#handling-reference-cycles):

- **Ordering and default.** Resolution MUST try in-Description identity
  matches, then a caller-supplied registry or cache, then network retrieval.
  Network retrieval MUST be disabled by default and MUST fail with a
  diagnostic identifying the unresolved reference rather than silently
  skipping it.
- **Atomicity.** The complete reference graph MUST be constructed and checked
  for cycles before any definitions are incorporated from it; a partial
  application MUST NOT be left in place after a graph is found invalid.
- **Duplicate identity.** A graph in which two distinct resources declare the
  same identity MUST be rejected as ambiguous, not resolved by source
  precedence.
- **Finite limits.** Tooling MUST enforce configurable upper bounds on at
  least: distinct retrieved resources, total decoded bytes, nesting depth,
  per-resource fan-out, decompression ratio, and redirects followed.

The limits are the part implementations skip and attackers find.

---

## What this would let us delete

| Proposal | Binding text it removes |
|---|---|
| 1. Dialect bindings | "Binding Parameters" — replaced by a reference; the JSON Structure parameter table stays. |
| 2. Classification ordering | "Reference Object Classification" in full. |
| 3. Unknown-dialect no-fallback | Two bullets of "Recognizing and Rejecting Dialects." |
| 4. URI matching | One bullet of "Recognizing and Rejecting Dialects." |
| 5. Embedded identity and extraction | "Default Resource Identity" and "Materializing Defaults for Standalone Processing" in full — the largest single block. |
| 6. Type determination contract | "Schema Inspection for Non-JSON Serializations" in full; the JSON Structure procedure stays. |
| 7. Third published schema | "Validating the Description Itself" in full. |
| 8. Non-OAD resources | One paragraph of "Resolving Cross-Document References." |
| 9. Conformance roles | The role definitions in "Conformance"; the JSON Structure specifics stay. |
| 10. Retrieval security | The ordering and limits text in "Resolving Cross-Document References" and most of "Security Considerations." |

Adopting all ten would reduce the binding to a dialect URI table, a type
system mapping, the reference and import rules peculiar to JSON Structure, and
a parameter table. Roughly a third of its current normative weight.

Adopting only 3, 6, and 7 would remove the cases where the absence of a rule
produces silently wrong output rather than inconsistent output. If appetite is
limited, those are the three.

## Sequencing

Proposals 3, 4, 8, and 10 are additive clarifications and appear
backward compatible; they constrain behavior that is currently undefined
rather than changing behavior that is currently defined. They are candidates
for 3.3.

Proposal 7 is a publishing change, not a specification change, and is already
under discussion in #4147. It could land independently of any of the others.

Proposals 1, 5, 6, and 9 are structural. Proposal 6 in particular requires
refactoring existing normative text in
[§4.24.4.2](https://spec.openapis.org/oas/v3.2.0.html#non-json-data),
and proposal 9 introduces a
vocabulary that would ripple through the document. These fit the Moonwalk
principle of loose coupling between "HTTP interfaces" and "content schema
formats" and may be better placed there — with the caveat that Moonwalk has
no planned end date, and these problems exist in shipping tooling today.

## References

- OpenAPI Specification 3.2.0, <https://spec.openapis.org/oas/v3.2.0.html>
  - [§4.1.2 OpenAPI Description Structure](https://spec.openapis.org/oas/v3.2.0.html#openapi-description-structure)
    · [§4.1.2.1 Parsing Documents](https://spec.openapis.org/oas/v3.2.0.html#parsing-documents)
    · [§4.1.2.2.1 Establishing the Base URI](https://spec.openapis.org/oas/v3.2.0.html#establishing-the-base-uri)
  - [§4.23 Reference Object](https://spec.openapis.org/oas/v3.2.0.html#reference-object)
    · [§4.24 Schema Object](https://spec.openapis.org/oas/v3.2.0.html#schema-object)
    · [§4.24.4.2 Non-JSON Data](https://spec.openapis.org/oas/v3.2.0.html#non-json-data)
    · [§4.24.7 Specifying Schema Dialects](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects)
  - [§6.5 Handling External Resources](https://spec.openapis.org/oas/v3.2.0.html#handling-external-resources)
    · [§6.6 Handling Reference Cycles](https://spec.openapis.org/oas/v3.2.0.html#handling-reference-cycles)
  - [Appendix F: Base URI Determination and Reference Resolution](https://spec.openapis.org/oas/v3.2.0.html#appendix-f-examples-of-base-uri-determination-and-reference-resolution)
    · [Appendix G: Parsing and Resolution Guidance](https://spec.openapis.org/oas/v3.2.0.html#appendix-g-parsing-and-resolution-guidance)
- OAS schema iterations, <https://spec.openapis.org/oas/>
- OAI issue #4147, "Improve 3.1+ schema `$id` UX"
- OAI issue #4152, "Make the 'latest' schema accessible programmatically"
- JSON Schema draft-06 release notes (draft-04 → draft-06 incompatibilities),
  <https://json-schema.org/draft-06/json-schema-release-notes>
- JSON Schema 2019-09 release notes (incompatible and semi-incompatible
  changes), <https://json-schema.org/draft/2019-09/release-notes>
- JSON Schema 2020-12 release notes (`items`/`prefixItems`, `contains`,
  `$dynamicRef`), <https://json-schema.org/draft/2020-12/release-notes>
- `draft-vasters-json-structure-oas-binding`, Part I
