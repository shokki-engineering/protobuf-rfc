# Decisions

What was decided and deliberately not built, kept because the schemas cite it.
`CLAUDE.md` and `docs/conventions.md` hold the rules; this holds the
consequences of applying them.

It is not a roadmap. The roadmap was `PLAN.md` and `plan/`, and both were
deleted when the last plan landed -- a finished plan is removed rather than
marked DONE, which is the convention those files themselves set.

## Deliberately not built

- **A SCIM-to-LDAP codec.** The contact lineage has RFC 9555, which maps vCard
  to JSContact element by element, so `runtime-go/rfc` follows a
  specification. **Nothing equivalent exists for SCIM and LDAP.** Writing one
  means inventing the mapping, which rule 14 permits only with every message
  saying so -- real design work with real choices, not a translation. Until
  someone needs it, the identity lineage has two models and no conversion, and
  that is stated rather than papered over.
- **RFC 9253's GAP parameter.** It attaches to RELATED-TO, and RELATED-TO is
  not modelled on components -- only `Alarm.snoozed_alarm_uid` uses that
  property, for the SNOOZE relation. GAP has nothing to attach to yet.
- **RFC 9554's PROP-ID, DERIVED and AUTHOR parameters.** They exist to make
  JSContact conversion lossless, and the conversion works without them. Add
  them when a round trip is shown to drop something they would have carried.
- **JSContact `localizations` (RFC 9553 section 2.7.1).** Typed
  `String[PatchObject]` -- a JSON-patch structure with no natural protobuf
  shape. It needs its own decision, not a guess.
- **RFC 9967, SCIM security event tokens.** A notification protocol rather
  than schema: it defines events a provider emits, which is a service concern
  and not a resource this repository would model.

- **JSCalendar, RFC 8984.** The canonical calendar model under rule 14, and
  deferred anyway. JSCalendar 2.0 obsoletes 8984 and has sat in IESG review
  for months without a number; the iCalendar↔JSCalendar conversion is also
  still a draft. Building 8984 now would ship a `v1` of a spec superseded
  before it landed. **Trigger: build when jscalendarbis is published**, to
  2.0, skipping 8984 entirely. Until then iCalendar is both the model and
  the format, and `docs/ontology.md` says so.

- **VTIMEZONE (§3.6.5).** Modelling a timezone database in protobuf when IANA
  publishes one, and `google.type.TimeZone` already carries the id and the
  tzdata version.
- **CalDAV capability negotiation, RFC 4791 §5.2.3-4**
  (`supported-calendar-component-set`, `supported-calendar-data`). What a
  collection *accepts*, as opposed to the limits it enforces. A resource
  calendar takes VEVENT and nothing else, which an adapter answers statically,
  so neither has earned a field. The four limits from §5.2.5-9 are built, on
  `Calendar` as `max_resource_size`, `min_date_time`, `max_date_time`,
  `max_instances` and `max_attendees_per_instance`.
- **WebDAV ACL and `CALDAV:read-free-busy`, RFC 3744 / RFC 4791 §6.1.** A
  generic permissions framework, not calendar data, and RFC 3744 is not in
  this schema's RFC list.
- **addressbook-multiget, RFC 6352 §8.7.** Repeated `GetVcard` today. Its
  calendar twin, `calendar-multiget` §7.9, has since been built as
  `Events.BatchGetEvents` because a CalDAV consumer needed the round trips;
  no consumer has asked the same of the address book, so this one waits.
- **PHOTO (§6.2.4) and KEY (§6.8.1) as fields.** Both are URI-valued, no
  different in shape from `Url` or `Geo` -- the risk is a `data:` URI
  embedding a full base64 blob inline, bloating every `ListVcards` page.
  Decided: each becomes its own sub-resource under `Vcard` when built (an
  `Avatar` for PHOTO, a separate credential resource for KEY, each with full
  AIP-131..135 CRUD), not AIP-157 partial responses, which would add
  `read_mask` plumbing across the whole service for two properties. Not
  built: no consumer yet, and a sub-resource is a real CRUD commitment
  (rule 8) for either one.
- **The remaining 40-odd RFC 4519 attribute types.** Adding them because they
  exist is exactly what rule 13 forbids.
- **`person` (§3.12) and `organizationalPerson` (§3.9).** `person` carries
  `sn`, `cn`, `userPassword`, `telephoneNumber`, `seeAlso`, `description` --
  a worse `Contact` than RFC 6350 already gives, and `Vcard` in `rfc6350/vcard/v1`
  already occupies the "addressable person" slot. A second person resource
  with no rule for choosing between them is worse than not having one; if
  LDAP-sourced people need representing, map them onto `Vcard` in the codec.
- **`country` (§3.2) and `locality` (§3.7) as resources.** `country` has one
  mandatory attribute (`c`, a two-letter ISO 3166 code) and `locality` has
  none. Neither has identity worth managing through CRUD -- a country is a
  value, not a resource. If a country code is needed it is a documented
  `string` field per rule 4, the same argument that removed the primitives
  layer.
- **`groupOfUniqueNames` (§3.6).** `Membership`'s own comment documents why:
  `uniqueMember` (§2.40) appends a UID to a DN so a reused name does not
  inherit an old membership, and `Membership.uid` solves that more directly.
- **RFC 8428 section 9's fragment identification.** The "rec=3,5" URI fragment
  that selects records by position, and RFC 8790 section 4 reuses it. A URI
  syntax for addressing into a document, not data -- rule 4's "an RFC defining
  a string format contributes a comment on a field, not a package". Selecting
  records by *content* is `PackRecords.SelectPackRecords`, which is what a
  caller actually wants; selecting the third record of a pack is a position
  that changes whenever the pack does.
- **A conversion method on `Units`.** RFC 8798 section 3 gives the arithmetic --
  multiply by scale, add offset -- and `Unit` carries both numbers as exact
  `google.type.Fraction`s, so a client converts locally. A `ConvertUnit` RPC
  would be a network round trip to do one multiplication, and rule 7 keeps
  resource services at the standard methods without a reason that applies to
  that resource alone.
- **Validating a record's `unit` against the `Units` registry.** RFC 8428
  section 4.5.2 permits a record with no unit at all and leaves anything it
  does not recognise to the application, so `Packs.CreatePack` never consults
  `Units`. Rejecting a pack for an unregistered unit would fail closed on the
  ordinary case of a registry copy lagging IANA.
- **Validating a pack's sensor names against `Sensors`.** Same shape: devices
  report before anyone catalogues them, so a pack naming an unregistered sensor
  is ordinary. The join is a string lookup on `senml_id`, in whichever
  direction the caller needs.

- **A published codec library.** `codec/go` is a conformance test and says so
  in its own module comment. Turning it into a product is a different project
  with a different support burden.


## Built after being deferred

Four things this file previously listed as not built now are, at the owner's
direction. Kept here because each was deferred with a reason, and the reason
is what changed:

- **`Sensor`** -- deferred because RFC 8428 section 4.7 declines to define a
  sensor and rule 4 says an RFC defining a string format earns no package. It
  is built and it is still native: every field on it is ours or RFC 6690's.
  What changed is where it sits. It is under `rfc8428/`, which **rule 5
  forbids for a native type**, so rule 5 now carries a written exception --
  *meaningless apart from that RFC* -- and `CLAUDE.md` records both the
  exception and the test that keeps it from spreading. `Project` is the
  control case: related to `Group`, and still in `shared/`.
- **`Stream`** -- deferred as a transmission mode rather than a resource. Built,
  and the argument that carried it is `StreamRecord`: section 4.8 resolves a
  relative time against when each *record* was sent, where a pack has one
  instant for all of them. Storing a stream as a pack collapses those and moves
  every reading in it. That is a data difference, not a delivery one.
- **`Unit`** -- deferred as a registry rather than data. Built, because RFC 8798
  section 4 asks for exactly this: the subregistry "is amenable to automatic
  interpretation, by making use of the scale and offset columns", and
  implementations SHOULD receive updates "via common repositories" instead of
  polling IANA. A deployment-owned, addressable copy is that repository. The
  enum this file rejected is still rejected -- `unit` stays a string.
- **RFC 8790's FETCH and PATCH** -- deferred as a transport concern duplicating
  `UpdatePack`. Built as `PackRecords`, and the duplication argument was wrong:
  a field mask names fields, so the finest it reaches is "replace all records",
  while RFC 8790 selects by *content*. Different addressing, not finer. It got
  no directory -- rule 6 gives those to defining RFCs, and 8790 refines Pack
  access -- and no method keeps its RFC name, because AIP-131 and AIP-134
  reserve `Fetch` and `Patch` as synonyms for `Get` and `Update`.

## The AIP-215 duplication tax

```mermaid
flowchart TB
    CT["CalendarTime<br/><i>one definition, five copies</i>"]

    subgraph copies["each package defines its own"]
        E["rfc5545/event"]
        TD["rfc5545/todo"]
        J["rfc5545/journal"]
        S["rfc5546/scheduling"]
        A["rfc7953/availability"]
    end

    CT -.-> E & TD & J & S & A
    E x-. "AIP-215 forbids<br/>importing it" .-x TD

    classDef ghost fill:none,stroke:#8250df,stroke-dasharray:4,color:#8250df
    class CT ghost
```

The dashed box is not a package — it is the type that *would* exist if one
were allowed. What exists is five identical definitions, and the crossed edge
is the import that would remove them.


`CalendarTime` is defined identically four times (`rfc5545/event`, `todo`,
`journal`, `rfc5546/scheduling`); `Classification` and `Participation` are
each duplicated too. One package per resource plus AIP-215's cross-package
ban means any value type shared between iCalendar components must be copied
per package -- `Recurrence`, `Attendee`, `Organizer` and `Alarm` are
deliberately *not* on `Todo` for the same reason.

Two ways out when a consumer forces the issue, neither free: copy the type
into each package that needs it (simple, but every codec then treats e.g.
`event.v1.Classification` and `todo.v1.Classification` as distinct types that
happen to agree), or collapse `rfc5545` into one package holding every
component (removes the duplication, breaks one-resource-per-package
independent versioning). Decide when something actually needs recurring
todos, not before.

## Open

Everything the plans covered is built. What remains is either a decision
nobody has needed to make yet or a consequence of work moving between
repositories.

- **`the-protobuf-project.org`** is the resource-type prefix on all fourteen
  `google.api.resource` declarations and is a placeholder domain. Cheap to
  change now; a breaking change once a consumer depends on the type strings.
- **Four committed files reference documents that are moving out.**
  `CLAUDE.md` line 183 *imports* `docs/ontology.md`, which is rule 14 itself;
  `README.md` and `docs/conventions.md` point at `PLAN.md` and `plan/`. Those
  break on a fresh clone until the documents land somewhere. Separately,
  `docs/references.md` still points at `plan/07-adding-data.md`, which is now
  `docs/adding-data.md` and has been stale for longer.
- **The AIP-215 duplication tax** (`CalendarTime` copied five times,
  `Classification` and `Participation` twice each) is unresolved by design --
  see the duplication tax above.

Two items that were open are not any more, recorded so they are not
rediscovered: `BUF_TOKEN` is set and `PublishModule` runs, which is where the
BSR module `runtime-go/rfc` depends on comes from; and the message-level CEL
rules are now executed, by `Validate()` in that module -- they had never been
run by anything, because `buf build`, `buf lint` and api-linter all pass on a
broken CEL expression.
