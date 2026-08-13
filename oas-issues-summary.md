# Gaps in OAS 3.2 Dialect Support, and Proposals to Close Them

**Status:** input for discussion with the OpenAPI Initiative.
**Target:** OAS 3.3, or 4.0 where a change is not backward compatible.
**Baseline:** OpenAPI Specification 3.2.0, published 19 September 2025.

OAS 3.1 shipped a schema-dialect extension point and specified only how to
**select** a dialect. Everything downstream of selection is undefined. This
document proposes thirteen fixes.

## Why it matters

**The gaps produce undefined behavior today, before any new dialect is
admitted.** OAS already sanctions dialects that are mutually incompatible with
each other — JSON Schema draft-04 through 2020-12, several of whose
transitions change validation outcomes silently. A `draft-07` Description is
legal under
[§4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects),
and it has no defined identity handling, no defined behavior in a tool that
has not implemented draft-07, and no published schema that validates it. See
[Why this is not a JSON Structure problem](#why-this-is-not-a-json-structure-problem)
for the evidence.

**One gap needs no second dialect at all.** Nothing in OAS says what a code
generator should name the types it emits, so the same Description yields
different type names in different tools (gap 13).

## The thirteen gaps at a glance

Gaps 1 to 7 are foundational and build on each other: what a binding is, what
object you are looking at, what unit carries a dialect, which dialect it is,
which reference layer applies, and what the resource is called. Gaps 8 to 13
are processing concerns that depend on the first seven being settled.

| # | Undefined today | What goes wrong |
|---|---|---|
| [1](#1-dialect-processing-is-undefined) | What it means to *process* a non-OAS-dialect Schema Object | Every binding invents its own contract, differently |
| [2](#2-reference-object-versus-schema-object-classification-is-not-ordered) | When Reference Object / Schema Object classification happens relative to dialect selection | A bare `$ref` is handed to a dialect processor that cannot read it |
| [3](#3-what-counts-as-a-dialect-selection-unit-is-undefined) | Which Schema Objects are the units a dialect attaches to | The boundary is defined by a JSON Schema keyword, so a dialect without it has none |
| [4](#4-unknown-dialects-have-no-defined-behavior) | What a tool does when it does not recognize the dialect URI | **Silently parsed as OAS dialect. Validation outcomes change with no error.** |
| [5](#5-dialect-uri-matching-semantics-are-unspecified) | Whether URI matching is exact, and how a tool comes to support a dialect at all | Near-misses match in one tool and not the next |
| [6](#6-the-two-reference-layers-are-not-separated) | Which reference layer a given reference belongs to | **A bundler inlines across the boundary and silently produces an invalid schema.** |
| [7](#7-an-embedded-schema-resource-without-an-identity-has-no-defined-identity) | The identity of a Schema Object with no `$id`; how identities are compared; what extraction means | A schema cannot be named, matched, or lifted out for standalone validation |
| [8](#8-type-determination-for-non-json-serializations-is-dialect-locked) | How to determine a value's type when the schema is not JSON Schema | **Non-JSON bodies cannot be parsed correctly. This breaks function, not tidiness.** |
| [9](#9-no-published-schema-validates-a-description-that-uses-another-dialect) | Which published schema validates a multi-dialect Description | `schema-base` rejects a Description that OAS itself permits |
| [10](#10-the-status-of-non-openapi-non-json-schema-documents-is-undefined) | The status of a type library pulled in by a dialect's own import mechanism | Undefined behavior for a mechanism OAS invited |
| [11](#11-there-is-no-conformance-role-model) | What "tooling" means when tools do disjoint jobs | "Is this tool OAS-compliant?" has no answer |
| [12](#12-external-retrieval-security-requirements-are-too-thin-to-implement) | Retrieval ordering, defaults, and limits | Every binding reinvents them, differently; the limits are what attackers find |
| [13](#13-there-is-no-scoping-framework-for-the-schemas-of-a-description) | The relationship between the name scopes of a Description's schemas | One Description, different generated type names per tool |

Three failure shapes. **Gaps 4, 6, and 8 fail silently** — wrong output, no
error raised at the point of damage. Gap 9 fails loudly but blocks: a
Description OAS permits will not validate against any published OAS schema.
The remainder surface as divergence, visible only once a second tool is
involved.

Each section below states what OAS says today, what it leaves undefined, and a
concrete proposal.

## Where this comes from

`draft-vasters-json-structure-oas-binding` binds JSON Structure into OpenAPI
Descriptions through the `$schema` / `jsonSchemaDialect` mechanism. Writing it
turned up an uncomfortable ratio: roughly two thirds of its normative
requirements have nothing to do with JSON Structure. They are answers to
questions any dialect binding must answer, and OAS does not answer them.

The binding therefore has two parts. Part I, "Dialect Binding Requirements,"
is dialect-agnostic and parameterized by a handful of declarations a binding
supplies. Part II binds JSON Structure to it. Part I exists only because OAS
defines dialect *selection* and stops there.

**The goal of this document is to make Part I unnecessary.** The
[deletion table](#what-this-would-let-us-delete) maps each proposal to the
text it would remove.

## Why it went unnoticed

[Section 4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects)
says a Schema Object may declare a dialect, that tooling MUST support the OAS
dialect, and that it MAY support others. The table above lists what is left
undefined from there.

That was survivable while the OAS dialect was the only dialect in practice.
With one dialect there is nothing to disambiguate: the two reference layers
blur together harmlessly because both are JSON Schema; the identity question
does not bite because nobody extracts schemas; the unknown-dialect question
never arises.

It bites the moment a second dialect exists. And if the next binding answers
these questions independently, it will answer them differently — at which
point a tool supporting two dialects has two contradictory conformance
obligations for the same code path. **Dialect bindings are where per-spec
divergence costs most, because the point of a binding is that one reader
handles all of them.**

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
| Type determination | The procedure yielding a value's JSON data type from the schema, for non-JSON serializations (see gap 8). |
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

## 3. What counts as a dialect-selection unit is undefined

**Today.**
[Section 4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects)
says `$schema` "MAY be present in any Schema Object that is a **schema
resource root**," and links that term to the JSON Schema Draft 2020-12
definition, which delimits a schema resource by `$id`.

**Gap.** The boundary that dialect selection attaches to is defined by a JSON
Schema keyword. A dialect with a different identity keyword, or with none at
all — which the binding declarations of gap 1 explicitly permit — has no
defined resource boundary in OAS, and therefore nothing for a dialect to
attach to.

Four further questions have no answer in the text:

- Which OAS positions hold a schema resource root. `components.schemas`
  entries plainly do. Whether an inline Schema Object at a request body,
  response, parameter, callback, or webhook position does is never stated.
- Whether a Schema Object nested inside an already-selected resource can open
  a second selection context.
- What a nested `$schema` means: override, error, or ignored.
- Whether selection is per root or per document, and therefore whether one
  Description may mix the OAS dialect with others.

The last one matters most in practice. **A Description that mixes dialects is
either the ordinary case or a malformed one, and OAS does not say which.**

**Proposal.** In
[§4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects):

1. Define "schema resource root" in dialect-neutral terms rather than by
   reference to JSON Schema's `$id`-based definition, and enumerate the OAS
   positions that hold one.
2. State that dialect selection happens per schema resource root, that a
   Description MAY contain Schema Objects in more than one dialect, and that
   tooling MUST apply the effective dialect independently at each root.
3. State that a Schema Object nested inside a selected resource is part of
   that resource and does not establish a second selection context; that a
   nested `$schema` MUST NOT override the root's dialect; and that tooling
   MUST reject a nested `$schema` that would make the nested object's dialect
   ambiguous.

## 4. Unknown dialects have no defined behavior

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

## 5. Dialect URI matching semantics are unspecified

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

State also how a tool comes to support an additional dialect at all. OAS says
tooling "MAY support additional values of `$schema`" without saying whether
that support is compiled in, configured out of band, or discovered by
retrieving the meta-schema. The distinction matters because a dialect may
permit deriving a custom meta-schema — one that carries its own URI while
incorporating a declared meta-schema's definitions — and no exact match can
anticipate such a URI. OAS should say that recognizing one is an explicit,
out-of-band configuration decision; that a tool so configured processes the
Schema Object under the dialect the derived meta-schema extends; and that a
tool not so configured applies the no-fallback rule of gap 4 rather than
guessing from the URI's shape or resolving it at read time.

## 6. The two reference layers are not separated

**Today.**
[Section 4.23](https://spec.openapis.org/oas/v3.2.0.html#reference-object)
defines the Reference Object and
[§4.24](https://spec.openapis.org/oas/v3.2.0.html#schema-object)
the Schema Object, whose `$ref` is JSON Schema's.
[Appendix F](https://spec.openapis.org/oas/v3.2.0.html#appendix-f-examples-of-base-uri-determination-and-reference-resolution)
specifies resolution for both. Every rule is written for a single referencing
mechanism.

**Gap.** A Schema Object in another dialect brings its own referencing
keywords, which need not be spelled `$ref`, need not mean what `$ref` means,
and need not be able to reach OAS component addresses at all. Two layers now
coexist in one document:

- **The OpenAPI layer.** An OAS `$ref` targeting a Schema Object as a whole,
  resolving per OAS, at any position in the Description.
- **The dialect layer.** Whatever referencing keywords the dialect defines,
  used *within* a Schema Object that has adopted that dialect.

OAS says nothing about which layer a given reference belongs to, or when that
determination is made, because with one dialect both layers are the same
machinery. **The concrete failure is a bundler that corrupts a valid
Description:** inlining an OAS `$ref` target into the body of a Schema Object
in another dialect yields a schema that is invalid in its own dialect, with
no error raised at the point of damage.

**Proposal.** State in
[§4.23](https://spec.openapis.org/oas/v3.2.0.html#reference-object) or
[§4.24.7](https://spec.openapis.org/oas/v3.2.0.html#specifying-schema-dialects):

1. Tooling MUST determine which reference layer a reference belongs to before
   resolving it, and that determination follows from the effective dialect of
   the containing schema resource.
2. An OAS `$ref` that targets a schema resource from outside it stays on the
   OpenAPI layer and is resolved by OAS rules unchanged — including where it
   sits inside a Schema Object that has not adopted a non-OAS dialect.
3. A reference inside the body of a resource that has adopted a dialect is
   governed by that dialect, and is valid only where the dialect permits it.
4. A reader MUST NOT reinterpret an OAS reference target as dialect content
   until that target has independently undergone dialect selection. Bundling
   and inlining MUST preserve this boundary.

## 7. An embedded schema resource without an identity has no defined identity

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
- Specify how identities are compared when matching a reference,
  distinguishing the **resource identity** — the declared value including any
  fragment, used as the base URI within that resource — from the **lookup
  key**, its fragment-free form, used only for matching. Require that two
  registered identities reducing to the same lookup key be rejected as
  ambiguous, and that the same resource identity registered twice be rejected.

## 8. Type determination for non-JSON serializations is dialect-locked

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

## 9. No published schema validates a Description that uses another dialect

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

## 10. The status of non-OpenAPI, non-JSON-Schema documents is undefined

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

## 11. There is no conformance role model

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

## 12. External retrieval security requirements are too thin to implement

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

## 13. There is no scoping framework for the schemas of a Description

**Today.** A Description holds many schema resources: every
[`components.schemas`](https://spec.openapis.org/oas/v3.2.0.html#components-object)
entry, and every inline Schema Object at a request body, response, parameter,
or header position. OAS gives each of them an identity, through `$self`,
`$id`, and JSON Pointer. It gives none of them a type identity, and it defines
no relationship between the name scopes they establish.

The omission is deliberate at the data-model layer:
[§4.24.4](https://spec.openapis.org/oas/v3.2.0.html#parsing-and-serializing)
places "class hierarchies" in the application form, "beyond the scope of this
specification." But
[§2](https://spec.openapis.org/oas/v3.2.0.html#introduction)
states that an OAD "can then be used by ... code generation tools to generate
servers and clients in various programming languages," and a code generator
has to name every type it emits.

The one name-inference rule OAS states is for XML nodes rather than types
([§4.26.3](https://spec.openapis.org/oas/v3.2.0.html#xml-node-names)): the
component key names a `components.schemas` entry, a property name names a
property schema, and "in all other cases ... no name can be inferred" and an
explicit name MUST be present. Component keys are the closest thing to a type
namespace OAS has, and their permitted syntax
([§4.7.1](https://spec.openapis.org/oas/v3.2.0.html#components-object):
`^[a-zA-Z0-9\.\-_]+$`, with `my.org.User` among the given examples) shows
authors already encoding namespaces into them by convention, with no meaning
assigned by the specification.

**Gap.** Two schema resources in one Description can define different types
under the same name. Neither is in the other's scope, so neither is wrong. The
collision exists only for a consumer that flattens the Description into a
single type space — which is what a code generator does.

This is the one gap in this document that needs no second dialect. Two
`components.schemas` entries can each define a `Pet` under `$defs`, with
different shapes, and nothing in the specification constrains what a generator
should call them. Each generator therefore picks its own answer, so one
Description yields different type names in different tools, and adding an
unrelated schema can rename an existing generated type.

The gap widens once other dialects are admitted, because a dialect can have
first-class named types and namespaces. JSON Structure requires a `name` on
every type and scopes `definitions` into a namespace hierarchy, so it can
express the resulting scopes exactly — but only once something establishes
that the resources are separate scopes at all.

**Proposal.**

1. State that each schema resource in a Description establishes a distinct
   name scope, and that the resources of one Description do not share a scope.
2. Require a consumer that aggregates the schemas of a Description into a
   single type space to preserve the resource boundary as a scope boundary,
   and to neither merge two resources' declarations into one scope nor rename
   a declaration to resolve a collision that arises only from merging.
3. Define a deterministic scope name per resource, along the lines the XML
   node-name rule already sets: the component key for a `components.schemas`
   entry, a name derived from the identifying parts of the position for an
   inline schema, and an error where neither yields one.
4. Add to the binding declarations of gap 1: what names a type in the dialect,
   and what constitutes one resource's scope. A dialect that names no types
   declares that, and the aggregation requirement does not apply to it.

Points 1 through 3 are dialect-neutral and would settle the OAS dialect's own
behavior. Point 4 is what carries the framework to the dialects the extension
point already admits.

---

## Why this is not a JSON Structure problem

**OAS already admits dialects that are mutually incompatible with each
other.**
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
schema." **JSON Schema mandates the no-fallback behavior that OAS leaves
undefined** (gap 4).

Four of the gaps above already produce undefined behavior for the drafts OAS
sanctions today:

- [Appendix F](https://spec.openapis.org/oas/v3.2.0.html#appendix-f-examples-of-base-uri-determination-and-reference-resolution)
  and [§4.1.2.1](https://spec.openapis.org/oas/v3.2.0.html#parsing-documents)
  name `$id` as the Schema Object's identity keyword.
  A draft-04 Schema Object declares `id`. OAS's base-URI machinery does not
  apply to a dialect OAS permits. (Gap 7.)
- A tool that has not implemented draft-07 has no defined behavior on
  encountering it. (Gap 4.)
- Nothing says whether `http://json-schema.org/draft-07/schema#` and
  `https://json-schema.org/draft-07/schema` select the same dialect. JSON
  Schema's own URIs vary in scheme and trailing `#` across drafts. (Gap 5.)
- `schema-base` pins `jsonSchemaDialect` to the OAS dialect, so a draft-07
  Description already fails it. (Gap 9.)

JSON Structure sits on the same axis: another dialect, selected by the same
keyword, assigning its own meanings to shared keyword spellings. Admitting it
introduces no category of risk that admitting draft-07 did not. **The
proposals above are not accommodations for JSON Structure. They are what the
existing extension point needs in order to work for the dialects OAS already
sanctions.**

---

## What this would let us delete

| Proposal | Binding text it removes |
|---|---|
| 1. Dialect bindings | "Binding Parameters" — replaced by a reference; the JSON Structure parameter table stays. |
| 2. Classification ordering | "Reference Object Classification" in full. |
| 3. Selection units | "Dialect Selection" in full. |
| 4. Unknown-dialect no-fallback | Two bullets of "Recognizing and Rejecting Dialects." |
| 5. URI matching and configured recognition | Two bullets of "Recognizing and Rejecting Dialects." |
| 6. Reference layers | "Reference Layer Separation" in full. |
| 7. Embedded identity, comparison, and extraction | "Default Resource Identity" and "Materializing Defaults for Standalone Processing" in full, plus the identity and lookup-key rules in "Resolving Cross-Document References" — the largest single block. |
| 8. Type determination contract | "Schema Inspection for Non-JSON Serializations" in full; the JSON Structure procedure stays. |
| 9. Third published schema | "Validating the Description Itself" in full. |
| 10. Non-OAD resources | One paragraph of "Resolving Cross-Document References." |
| 11. Conformance roles | The role definitions in "Conformance"; the JSON Structure specifics stay. |
| 12. Retrieval security | The ordering and limits text in "Resolving Cross-Document References" and most of "Security Considerations." |
| 13. Scoping framework | "Type Identity and Scope" in full; the JSON Structure namespace construction stays. |

Adopting all thirteen would reduce the binding to a dialect URI table, a type
system mapping, the reference and import rules peculiar to JSON Structure, and
a parameter table. Roughly a third of its current normative weight.

## Change cost

Gaps 4, 5, 6, 10, and 12 constrain behavior that is currently undefined rather
than changing behavior that is currently defined. They are additive and appear
backward compatible.

Gap 9 is a publishing change rather than a specification change, and is already
under discussion in #4147.

Gaps 1, 3, 7, 8, 11, and 13 require structural edits. Gap 3 redefines a term
OAS currently delegates to JSON Schema. Gap 8 means refactoring existing
normative text in
[§4.24.4.2](https://spec.openapis.org/oas/v3.2.0.html#non-json-data).
Gap 11 introduces a vocabulary that ripples through the document. Gap 13
changes output authors see today.

That structural set fits the Moonwalk principle of loose coupling between
"HTTP interfaces" and "content schema formats." Moonwalk has no planned end
date; these problems exist in shipping tooling now.

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
