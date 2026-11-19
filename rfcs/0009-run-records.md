---
rfc: 0009
title: Run records
status: Draft # Draft | Active | Accepted | Rejected | Superseded
authors:
  - Louis-Félix Nothias (@lfnothias)
created: 2026-11-18
tracking-issue: # to open with the "RFC proposal" template
superseded-by:
---

<!--
Drafting notes for the author (delete before opening the pull request):
- Written against astra-spec origin/main f59492c (2026-11-14), which holds
  RFCs 0001-0008, `provenance.yaml` and `types.yaml`. The local checkout at
  a local checkout is at 580802f (v0.0.14+1) and predates all of
  them: rebase the branch on origin/main before adding this file.
- 0009 is the next free number on origin/main. Add a row to rfcs/README.md.
- Statements marked "(unverified)" were not checked against their source.
- 2026-11-19: the six unresolved questions answered (time names mapped,
  `commit` optional with `universe`, `usage_by_model`, records kept outside
  the repository with `generated_by.run_content_hash`, `cost.price_list`,
  actor `same_as`); see *Resolved in drafting*.
-->

## Context

ASTRA records what an analysis is and, since [RFC 0005](0005-result-provenance-and-layout.md),
how each result was made: a provenance record per result, whose `generated_by`
names the commit, the inputs read, the environment, the runner and when the
recipe started and finished. That covers a recipe that makes a file.

It does not cover a **run** that is not one recipe making one result, and
most work done by agents is of that kind:

- an agent session that edits code, calls tools and models for an hour, and
  makes several results or none;
- one model call that judges something and writes no file at all;
- a build that runs a dozen model-driven stages, each with its own model.

For these nothing in ASTRA says who acted (a person, an agent or a tool, and
on whose behalf), how the run ended (an exit code, a timeout), how many model
calls it made, how many tokens it used, or what it cost and whether that cost
was measured or estimated. Tools that need these fields today put them where
the schema lets them, in free text.

**Evidence: free tags in an ASTRA export.** AgenticScienceBuilder (ASB) writes
`astra/run_trace.yaml` next to the ASTRA description of a paper, describing the
build that produced it
(`AgenticScienceBuilder/src/agentic_science_builder/astra_export/run_trace.py`).
The build's configuration fits ASTRA well, one decision per model and per flag,
so that two builds that differ only in one model are two universes over one set
of decisions. Everything else about the run is a tag on the analysis or on a
sub-analysis per stage (tag names from the exporter; values illustrative):

```yaml
tags:
  - asb:build_id=build_abc123
  - asb:pipeline_health=ok
  - asb:calls=212
  - asb:tokens_in=1840211
  - asb:tokens_out=93117
  - asb:cost_usd=11.42
  - asb:duration_seconds=1834.2
  - asb:git_sha=4015b57a
  - asb:git_dirty=False
# and on each model decision: asb:prompt_version=v4, asb:seed=42, asb:temperature=0.2
```

Tags carry no type, no unit and no difference between "unknown" and zero.
ASB's own call logger turns a missing token count into `0` (`int(... or 0)`)
and records `0.0` as the cost of calls routed through a subscription harness
(`llm_logging.py`), so a sum of these tags reports zero for spending that was
not measured. The export also targets ASTRA 0.0.12 (`narrative`, `Analysis.id`,
`version`), which v0.1 no longer accepts, so it has to move anyway.

**Three consumers need the same fields.**

- **ASB build runs.** Per build and per stage: who ran, on which model, when,
  how it ended, calls, tokens and cost. Read by ASB's renderer and run viewer.
- **A lab tool recording agent runs and judge calls.** A member's agent works
  on the member's behalf; a judge model checks what the member shares and may
  fail or time out, which still costs tokens. Each needs an actor kind, times,
  outcome, tokens with unknowns kept apart from zeros, and digests of what was
  read, without any of the member's text. (Collegium, a lab tool built in the
  same group, is the first; its field design names these fields.)
- **indicium's trace profile.** Its experimental research-memory profile v2
  defines an `execution_receipt` on an executed activity with exactly the keys
  `executor, code, configuration, environment, inputs, outputs, started_at,
  finished_at, exit_code, model_calls, tokens{input, output},
  cost{amount, currency, basis}` and an actor registry with kinds `human`,
  `agent`, `tool`, `unknown`. It records these as declarations bound to cited
  sources; it has no place to say which analysis and which decisions a run
  was configured with. (The profile is on an unreleased indicium branch.)

The gap is narrow. RFC 0005 already records a run of a recipe as a PROV
activity with a runner, an environment and two timestamps. This RFC adds the
record for a run of anything else, using the same types and, where indicium
already names a field with the same meaning, the same name (except the two
times, which keep the names RFC 0005 gave them and are mapped).

Prior art considered below: W3C PROV-O (which ASTRA records already map to),
the RO-Crate Workflow Run profiles, and the OpenTelemetry semantic conventions
for generative AI.

## Proposal

In plain language: **whatever runs something for an analysis may write a run
record: one small JSON file saying who ran what, under which decisions, when,
how it ended, what it used and what it cost, with references and content
hashes for the code, configuration, environment, inputs and outputs, and never
their content.** A run record is optional. It changes nothing in `astra.yaml`,
and a result's provenance record can name the run that made it.

### Guide-level explanation

A judge model is asked to check something and times out after two minutes.
It made one call and was billed for its input; nobody knows the price:

```json
{
  "spec_version": "0.2.0",
  "id": "judge_0007",
  "actors": {"agent:judge": {"kind": "agent", "label": "Lesson judge"}},
  "executor": "agent:judge",
  "execution": "failed",
  "stop_reason": "timeout",
  "started": "2026-11-24T10:40:02.118Z",
  "finished": "2026-11-24T10:42:02.120Z",
  "models": ["example-provider/model-x"],
  "model_calls": 1,
  "tokens": {"input": 18211, "output": 0}
}
```

- `execution` says how the run ended; the judge call has no process, so it
  has no `exit_code`, and none is invented.
- There is no `cost`: the cost is unknown, and an unknown is left out, never
  written as `0`. `output: 0` is a count the provider reported.
- The record says nothing about what the judge read or said. A verdict is a
  judgement about something else, and belongs in a tool that records
  judgements (see *Boundary with indicium*).

A longer run that ran code under a universe, on someone's behalf, is the
conformance example under *Examples*.

### 1. Where run records go

A run belongs to the analysis whose decisions it was configured with. Its
record may be committed in that analysis's repository or kept elsewhere, such
as in a tool's database or a hub; the format and every rule apply wherever it
is kept. A record committed in the analysis's folder MUST be at
`.astra/runs/{id}.json`, where `{id}` is the run's `id`. `.astra/` is already
reserved by RFC 0005 (`RESERVED_PATH`), so no input can collide with it. A
record kept elsewhere is still identified by its `id`.

Committed run records are tracked in git like provenance records. Nothing
requires a run record, nor committing one: no rule says a run must have one,
and a checker never reports a result `missing` for lack of one.

A run record that another record cites, a provenance record through
`generated_by.run` or an indicium source (section 4), MUST be reachable by
whoever reads the citing record: committed beside it, or kept where that
reader can fetch it. It SHOULD be cited with the SHA-256 of its bytes, so that
a reader holding a copy from anywhere can check it is the record cited.
`part_of` carries no hash: a larger run's record is often written after its
parts'.

A run that is part of a larger run (a stage of a build, one call inside a
session) has its own record and names the larger one in `part_of`. Records
stay flat; a tree of runs is read from `part_of`.

A provenance record MAY name the run that made its result in a new optional
field, `generated_by.run`, holding the run's `id`, and SHOULD then give the
SHA-256 of the run record's bytes in a second optional field beside it,
`generated_by.run_content_hash`. These two fields are the one change to RFC
0005's record; nothing else in RFC 0005 changes.

### 2. What a run record holds

A new schema file `run.yaml` sits beside `analysis.yaml` and `provenance.yaml`.
Through it a run record maps onto W3C PROV-O, as a provenance record does.

Required: `spec_version`, `id`, `actors`, `executor` and `execution`. The
times are recommended, as `Output.format` was in 0.0.x: a validator warns
when they are missing. Everything else is optional.

| Field | Meaning |
|---|---|
| `spec_version` | *required* - the version of the ASTRA specification the record follows |
| `id` | *required* - the run's id, unique in its analysis; the stem of its file name when the record is committed |
| `part_of` | the `id` of the run this run is part of, in the same analysis |
| `commit` | the full id of the commit the run's code and `astra.yaml` were checked out at, with no change to tracked files; left out when the run came from a working tree with changes or was imported, and then `universe` and `decisions` are read at the commit that adds the record |
| `universe` | the id of the universe of the analysis the run was configured with |
| `decisions` | options the run selected, as `<decision>: <option>`, written over the universe's effective selection decision by decision (as a universe's own `decisions` are over the one it is `derived_from`), or on their own when there is no `universe` |
| `actors` | *required* - who took part, keyed by actor id, each with a `kind`: `human`, `agent`, `tool` or `unknown`, and optionally a `label`, a `version` and a `same_as` IRI naming the same actor across analyses |
| `executor` | *required* - the actor id of who ran it |
| `on_behalf_of` | the actor id of who the executor acted for |
| `started`, `finished` | *recommended* - when the run started and finished, in UTC to the millisecond, ending in `Z` |
| `execution` | *required* - how it ended: `executed`, `failed` or `partial` |
| `exit_code` | the process exit code, where there was a process |
| `stop_reason` | a lowercase token saying why it stopped, such as `completed`, `timeout`, `budget`, `refused`, `killed` |
| `models` | the models called, as the provider names them |
| `model_calls` | how many model calls were made, failed ones included |
| `tokens` | `input` and `output`, both required when `tokens` is given, and optionally `reasoning`, `cache_read`, `cache_write` |
| `cost` | `amount`, `currency` (ISO 4217) and `basis`: `measured` (billed or reported for this run) or `estimated` (computed from counts and a price list), and optionally `price_list`, a reference to the price list an estimate used |
| `usage_by_model` | the run's usage broken down by model: entries each with a `model`, and optionally `model_calls`, `tokens` and `cost` for that model alone |
| `code`, `configuration`, `environment`, `inputs`, `outputs` | references, each a `ref` with an optional `content_hash` and `size` |
| `data_origin` | whether what the run worked on is `real`, `synthetic` or `mixed` |
| `event_log` | a reference to a log of the run's events (tool calls, model calls) kept outside the record |

**Tokens.** `input` counts every input token the run's model calls processed,
cached ones included; `cache_read` and `cache_write` say how many of those were
read from or written to a provider's cache. `output` counts every output token;
`reasoning` says how many of those were reasoning tokens. So
`cache_read + cache_write <= input` and `reasoning <= output`, and a consumer
that only knows input and output (indicium's receipt) loses the breakdown but
never the totals. Providers count differently: a writer converts to these
meanings or leaves the field out.

**References.** A `ref` is a path relative to the analysis folder with `/`
between folders, an ASTRA reference (`output1#baseline`, so an output points at
a result and its provenance record), or an IRI (`urn:...`). It never starts
with `/`. `content_hash` is the SHA-256 of the bytes `ref` names, as everywhere
in ASTRA, never of a stand-in, never of metadata about them.

**Unknown and zero.** As everywhere in ASTRA, no field is ever `null`
(`WRONG_TYPE`): an unknown value is left out. A `0` is an observed zero. A
writer MUST NOT write `0` for a count or an amount it did not observe
(`UNKNOWN_AS_ZERO`). A call routed through a subscription has no per-call
price: its `cost` is left out, not `0`.

**Without `commit`.** A run record without `commit` records a run that came
from a working tree with changes to tracked files, or one imported from
another store. It may still set `universe` and `decisions`; they are then read
at the commit that adds the record to the repository. A record that has no
`commit` and is kept outside the repository has no such commit, so it sets
neither.

**Usage by model.** A run that called several models may break its usage down
in `usage_by_model`: one entry per model, each with its `model` (one of
`models`) and, for that model alone, `model_calls`, `tokens` and `cost` of the
same shape and rules as the run's. An entry leaves out what was not observed
for it (`UNKNOWN_AS_ZERO`). Where an entry and the run both give a quantity
and every entry gives it, the entries sum to the run's (`USAGE_SUM_MISMATCH`);
costs are summed only when they share one `currency` and one `basis`. The
run's `cost` is the cost of the whole run, so when one entry's cost is unknown
the run's is left out too. The breakdown is for usage only. Work that is a run
of its own, with its own times, outcome, executor, inputs or outputs, such as
a stage of a build, gets its own record with `part_of`.

**Price lists.** An `estimated` cost SHOULD name the price list it was
computed from in `price_list`, a reference with a `content_hash` when the list
is a file, so that two estimates can be compared (`MISSING_PRICE_LIST`, a
warning; the record stays valid).

#### Schema

`src/astra/schema/run.yaml`:

```yaml
---
id: https://w3id.org/astra/run
name: run
description: |-
  The run record: what an executor writes for one run done for an analysis,
  saying who ran it, under which decisions, when, how it ended, what it used
  and what it cost, with references to what it read and made. Through this
  schema a record maps onto W3C PROV-O.
# Placeholder version. The real version is injected at build time: the tag at
# HEAD, else `dev` (or RELEASE_VERSION) — see `_stage-versioned` in the justfile.
version: 0.0.0
license: https://creativecommons.org/licenses/by/4.0/

prefixes:
  astra: https://w3id.org/astra/
  linkml: https://w3id.org/linkml/
  prov: http://www.w3.org/ns/prov#
  schema: http://schema.org/

default_prefix: astra
default_range: string

imports:
  - linkml:types
  - types
  - analysis

enums:

  ActorKind:
    description: What kind of actor took part in a run.
    permissible_values:
      human:
        description: A person.
      agent:
        description: >-
          Software that chooses its own actions, such as a model-driven agent
          or a model answering a call.
      tool:
        description: >-
          Software that does what it is told, such as a runner, a harness or
          a script.
      unknown:
        description: The writer doesn't know which of these it is.

  Execution:
    description: How a run ended.
    permissible_values:
      executed:
        description: It ran to its end and did what it was run for.
      failed:
        description: It stopped without doing what it was run for.
      partial:
        description: It stopped having done part of what it was run for.

  DataOrigin:
    description: Where what a run worked on comes from.
    permissible_values:
      real:
        description: Real data or material.
      synthetic:
        description: Made up for the purpose, such as a simulation or a fixture.
      mixed:
        description: Some of each.

  CostBasis:
    description: How a cost was obtained.
    permissible_values:
      measured:
        description: Billed or reported by the provider for this run.
      estimated:
        description: Computed from counts and a price list.

classes:

  RunRecord:
    tree_root: true
    class_uri: prov:Activity
    description: >-
      One run done for an analysis, which its executor writes as JSON: at
      `.astra/runs/{id}.json` in the analysis's folder when it is committed,
      or wherever else it is kept. Read as RDF, its top-level node stands for
      the run.
    attributes:
      spec_version:
        range: spec_version
        required: true
        description: >-
          The version of the ASTRA specification the record follows, such as
          `0.2.0`.
      id:
        range: id
        required: true
        description: >-
          The run's id, unique among the runs of its analysis, such as
          `run_5d2a9c71e04b8f36`: the stem of the record's file name when it
          is committed, and what names it wherever else it is kept.
      part_of:
        range: id
        slot_uri: prov:wasInformedBy
        description: >-
          The id of the run this run is part of, recorded in the same
          analysis, such as a build a stage belongs to.
      commit:
        range: commit_id
        description: >-
          The full id of the git commit the run's code and `astra.yaml` were
          checked out at, with no change to tracked files. A run from a
          working tree with such changes, or one imported from another
          store, leaves it out.
      universe:
        range: id
        description: >-
          The id of the universe of the analysis the run was configured
          with: an entry of its `universes`, never an import, at the record's
          `commit`, or at the commit that adds the record when it has none.
      decisions:
        range: DecisionSelection
        multivalued: true
        inlined: true
        description: >-
          Options the run selected, as `<decision>: <option>`, keyed as a
          universe's `decisions` are. With a `universe`, they are written over
          its effective selection decision by decision; without one, they are
          the run's whole selection.
      actors:
        range: Actor
        multivalued: true
        inlined: true
        required: true
        minimum_cardinality: 1
        description: Who took part in the run, keyed by actor id.
      executor:
        range: actor_id
        required: true
        slot_uri: prov:wasAssociatedWith
        description: The actor id, a key of `actors`, of who ran it.
      on_behalf_of:
        range: actor_id
        description: >-
          The actor id, a key of `actors`, of who the executor acted for. It
          says who the run was for, not that it was authorised.
      started:
        range: timestamp
        recommended: true
        slot_uri: prov:startedAtTime
        description: >-
          When the run started, in UTC to the millisecond, such as
          `2026-11-25T14:00:00.000Z`.
      finished:
        range: timestamp
        recommended: true
        slot_uri: prov:endedAtTime
        description: >-
          When the run finished, in UTC to the millisecond, such as
          `2026-11-25T15:12:41.503Z`.
      execution:
        range: Execution
        required: true
        description: How the run ended.
      exit_code:
        range: integer
        description: >-
          The exit code of the run's process. Left out where there was no
          process, such as a single model call.
      stop_reason:
        pattern: '^[a-z0-9][a-z0-9_.+-]*$'
        annotations:
          rule: INVALID_VALUE
        description: >-
          Why the run stopped, as a lowercase token, such as `completed`,
          `timeout`, `budget`, `refused` or `killed`. The value space is open.
      models:
        multivalued: true
        minimum_cardinality: 1
        description: >-
          The models the run called, as their provider names them, such as
          `example-provider/model-x`.
      model_calls:
        range: integer
        minimum_value: 0
        description: How many model calls the run made, failed ones included.
      tokens:
        range: Tokens
        inlined: true
        description: The tokens the run's model calls used.
      cost:
        range: Cost
        inlined: true
        description: >-
          What the whole run cost. Left out when the cost of any part of it
          is unknown.
      usage_by_model:
        range: ModelUsage
        multivalued: true
        inlined_as_list: true
        minimum_cardinality: 1
        description: >-
          The run's usage broken down by model, one entry per model of
          `models`.
      code:
        range: Reference
        multivalued: true
        inlined_as_list: true
        minimum_cardinality: 1
        slot_uri: prov:used
        description: The code the run ran, such as a harness at a version.
      configuration:
        range: Reference
        multivalued: true
        inlined_as_list: true
        minimum_cardinality: 1
        slot_uri: prov:used
        description: >-
          The configuration the run read beyond its decisions, such as a
          settings file or a prompt template, by reference and hash only.
      environment:
        range: Reference
        multivalued: true
        inlined_as_list: true
        minimum_cardinality: 1
        slot_uri: prov:used
        description: >-
          The environment the run ran in, such as a container image or a
          lock file.
      inputs:
        range: Reference
        multivalued: true
        inlined_as_list: true
        minimum_cardinality: 1
        slot_uri: prov:used
        description: What the run read.
      outputs:
        range: Reference
        multivalued: true
        inlined_as_list: true
        minimum_cardinality: 1
        slot_uri: prov:generated
        description: What the run made.
      data_origin:
        range: DataOrigin
        description: >-
          Whether what the run worked on is real, synthetic or mixed. Left out
          when unknown.
      event_log:
        range: Reference
        inlined: true
        description: >-
          A log of the run's events, such as its tool calls, kept outside the
          record.
    rules:
      - description: A run that executed and has an exit code has exit code 0.
        annotations:
          rule: EXIT_CODE_MISMATCH
        preconditions:
          slot_conditions:
            execution:
              equals_string: executed
            exit_code:
              value_presence: PRESENT
        postconditions:
          slot_conditions:
            exit_code:
              equals_number: 0

  Actor:
    class_uri: prov:Agent
    description: A person or a piece of software that took part in a run.
    attributes:
      id:
        key: true
        range: actor_id
        description: The actor's id, written as its key in `actors`.
      kind:
        range: ActorKind
        required: true
        description: Whether the actor is a human, an agent or a tool.
      label:
        description: A short display name.
      version:
        slot_uri: schema:softwareVersion
        description: For software, its version, such as `1.4.2`.
      same_as:
        range: iri
        slot_uri: schema:sameAs
        description: >-
          An absolute IRI naming the same actor outside this record, so that
          it can be recognised across analyses, such as an ORCID IRI for a
          person or a URL for a release of a tool.

  ModelUsage:
    description: >-
      The part of a run's usage that went to one model. Each field has the
      shape and the rules of the run's field of the same name, and is left
      out when it was not observed for this model.
    attributes:
      model:
        required: true
        description: >-
          The model, as its provider names it: one of the run's `models`.
      model_calls:
        range: integer
        minimum_value: 0
        description: How many calls the run made to this model, failed ones included.
      tokens:
        range: Tokens
        inlined: true
        description: The tokens this model's calls used.
      cost:
        range: Cost
        inlined: true
        description: What this model's calls cost.

  Tokens:
    description: >-
      Token counts summed over a run's model calls. `input` includes cached
      tokens; `cache_read` and `cache_write` are parts of it. `reasoning` is a
      part of `output`.
    attributes:
      input:
        range: integer
        minimum_value: 0
        required: true
        description: Every input token processed, cached ones included.
      output:
        range: integer
        minimum_value: 0
        required: true
        description: Every output token, reasoning tokens included.
      reasoning:
        range: integer
        minimum_value: 0
        description: The output tokens that were reasoning tokens.
      cache_read:
        range: integer
        minimum_value: 0
        description: The input tokens read from a provider's cache.
      cache_write:
        range: integer
        minimum_value: 0
        description: The input tokens written to a provider's cache.

  Cost:
    description: What a run cost, and how that was obtained.
    attributes:
      amount:
        range: float
        minimum_value: 0
        required: true
        description: The amount, in `currency`.
      currency:
        range: currency_code
        required: true
        description: The ISO 4217 code of the currency, such as `EUR` or `USD`.
      basis:
        range: CostBasis
        required: true
        description: Whether the amount was measured or estimated.
      price_list:
        range: Reference
        inlined: true
        description: >-
          The price list an `estimated` amount was computed from, such as a
          provider's published prices on a date, with the hash of its bytes
          when it is a file.

  Reference:
    class_uri: prov:Entity
    description: >-
      Something a run used or made, named by reference, with the hash and size
      of its bytes when known. Never its content.
    attributes:
      ref:
        required: true
        pattern: '^[^/]'
        annotations:
          rule: INVALID_PATH
        description: >-
          A path relative to the analysis's folder, with `/` between folders;
          an ASTRA reference, such as `output1#baseline`; or an IRI. It is
          non-empty and doesn't start with `/`.
      content_hash:
        range: content_hash
        description: >-
          The SHA-256 digest of the bytes `ref` names, never of something
          standing in for them.
      size:
        range: integer
        minimum_value: 0
        description: The size of those bytes.
```

Additions to `src/astra/schema/types.yaml`:

```yaml
  actor_id:
    typeof: string
    description: >-
      The id of an actor in a run record: an id, then any number of `:` and a
      segment of lowercase ASCII letters, digits, `_`, `.` and `-`, such as
      `member_a`, `agent:judge` or `tool:harness`.
    structured_pattern:
      syntax: '^{id}(:[a-z0-9][a-z0-9_.-]*)*$'
      interpolated: true
      partial_match: true
    annotations:
      rule: INVALID_ID

  currency_code:
    typeof: string
    description: An ISO 4217 currency code, three uppercase ASCII letters, such as `EUR`.
    pattern: "^[A-Z]{3}$"
    annotations:
      rule: INVALID_VALUE
```

Additions to `Generation` in `src/astra/schema/provenance.yaml`:

```yaml
      run:
        range: id
        slot_uri: prov:wasInformedBy
        description: >-
          The id of the run during which the recipe ran, a run of the
          analysis the result belongs to, whose record is at
          `.astra/runs/{id}.json` in that analysis's folder or wherever else
          the reader can reach it.
      run_content_hash:
        range: content_hash
        description: >-
          The SHA-256 digest of the bytes of the run record `run` names.
          Given only with `run`.
```

`run_content_hash` is the one addition this RFC proposes to RFC 0005's record
beyond `run` itself: a sibling field rather than a change to `run`'s range, so
that `run` stays an id and a record without the hash stays valid.

### 3. Rules

New rows for the rules table of the specification. Codes in **bold** are new;
the others extend an existing code to run records.

| Code | Level | Rule |
|---|---|---|
| **parse** | | |
| `INVALID_SYNTAX` | MUST | A run record is UTF-8 JSON text, no object repeating a name |
| **schema** | | |
| `MISSING_FIELD` | MUST | A run record has `spec_version`, `id`, `actors`, `executor`, `execution`; an actor, `kind`; `tokens`, `input` and `output`; `cost`, `amount`, `currency`, `basis`; an entry of `usage_by_model`, `model`; a reference, `ref`; a `generated_by` with `run_content_hash`, `run` |
| `MISSING_FIELD` | SHOULD | A run record has `started` and `finished`; a `generated_by` with `run` has `run_content_hash` |
| `WRONG_TYPE` | MUST | (as today) no value is `null`; an unknown is left out |
| `INVALID_VALUE` | MUST | `execution`, `kind`, `data_origin`, `basis` are among their values; `currency` is three uppercase letters; `stop_reason` matches `^[a-z0-9][a-z0-9_.+-]*$`; an actor's `same_as` is an absolute IRI; `run_content_hash` is a content hash; counts, `size` and `amount` are ≥ 0 |
| `INVALID_ID` | MUST | `id`, `part_of` and `universe` are ids; actor keys, `executor` and `on_behalf_of` are actor ids |
| `INVALID_PATH` | MUST | A reference's `ref` is non-empty and doesn't start with `/` |
| **`EXIT_CODE_MISMATCH`** | MUST | A run `executed` with an `exit_code` has `0` |
| **check** (a broken rule makes the run record `invalid`) | | |
| **`MISPLACED_RUN`** | MUST | A run record committed in an analysis's folder is at `.astra/runs/{id}.json`, its `id` the stem of its file name; a record kept elsewhere is found by its `id` |
| **`EXIT_CODE_MISMATCH`** | MUST | A run `failed` with an `exit_code` has one other than `0` |
| **`UNKNOWN_ACTOR`** | MUST | `executor` and `on_behalf_of` are keys of `actors` |
| **`INVALID_INTERVAL`** | MUST | `finished` is not before `started` |
| **`INVALID_TOKEN_COUNTS`** | MUST | `cache_read + cache_write <= input` and `reasoning <= output`, in `tokens` and in each entry's `tokens` |
| **`UNKNOWN_MODEL`** | MUST | Each `model` of `usage_by_model` is one of `models`, and no two entries have the same `model` |
| **`USAGE_SUM_MISMATCH`** | MUST | Where the run gives `model_calls`, a count of `tokens` or `cost.amount`, and every entry of `usage_by_model` gives it too, the entries' values sum to the run's; costs only when every entry's `currency` and `basis` are the run's |
| **`UNKNOWN_PARENT_RUN`** | MUST | `part_of` names a run record of the same analysis, and no chain of `part_of` loops |
| **`UNKNOWN_RUN_UNIVERSE`** | MUST | `universe` names a universe, not an import, of the analysis at the record's `commit`; a record without `commit` (from a working tree with changes, or imported) is read at the commit that adds it, and one with neither sets no `universe` |
| `UNRESOLVED_ADDRESS`, `UNKNOWN_OPTION` | MUST | Each key of `decisions` names a decision, and each value one of its options, read at the same commit as `universe`, also when there is no `universe`; a record with neither commit sets no `decisions` |
| **`MISSING_PRICE_LIST`** | SHOULD | A `cost` with `basis: estimated` has a `price_list` (a warning; the record stays valid) |
| **`MISSING_RUN`** | MUST | The run record a provenance record's `generated_by.run` names is reachable by the reader: at `.astra/runs/{id}.json` in its analysis, or where the reader is told run records are kept (reported against the provenance record; the result's status doesn't change) |
| **`RUN_HASH_MISMATCH`** | MUST | A `generated_by.run_content_hash` is the SHA-256 of the bytes of the run record `run` names (reported against the provenance record) |
| **writer** (whatever writes a run record: a runner, a harness, an exporter) | | |
| `FALSE_RECORD` | MUST | Every field of a run record its writer writes is true of the run |
| **`UNKNOWN_AS_ZERO`** | MUST | A writer leaves out a count, size or amount it did not observe, never writing `0` for it, in the run's fields and in each entry of `usage_by_model` |
| **`CONTENT_IN_RUN_RECORD`** | SHOULD | No field holds a prompt, a model or tool payload, or text written by a person; `label` and `stop_reason` are short and generic |

A run record never becomes `stale`: it is the history of something that
happened. Whether the bytes behind its references have changed since is a
question a tool may ask by rehashing them (indicium does, for its receipts);
this RFC does not define a status for it.

### 4. Boundary with indicium

ASTRA records **what ran**: under which analysis and decisions, by whom,
when, how it ended, what it used and what it cost, and references with hashes
to what it read and made. indicium records **what people and agents claim
and judge** about it: observations, claims and hypotheses, reviews and their
verdicts, a curator's selection under a mandate, and the qualifications of a
trace (method, scope, coverage). The two meet at three points, and neither
copies the other:

- **One id.** An ASTRA run id is a valid indicium record id (indicium accepts
  any non-empty string), so the indicium `activity` record for a run takes the
  run's `id`. The reverse doesn't hold: an indicium id is not always an ASTRA
  id.
- **The run record as a source.** The activity cites the run record as an
  indicium source of kind `artifact`, with the SHA-256 of the record's bytes
  and where to reach it: `path: .astra/runs/<id>.json` when the record is
  committed, else the place it is kept (a tool's database, a hub), which the
  reader must be able to reach (section 1). Its `execution_receipt` is computed from
  the run record field by field (table below), so the record is the
  authority and the receipt a projection of it.
- **Shared names.** Where a field means the same thing, it has the same name:
  `executor`, `code`, `configuration`, `environment`, `inputs`, `outputs`,
  `exit_code`, `model_calls`, `tokens.input`, `tokens.output`, `cost.amount`,
  `cost.currency`, `cost.basis` and its values, `execution` and its values,
  `data_origin` and its values, actor `kind` and its values. The two times
  keep their own names on each side, `started` and `finished` here,
  `started_at` and `finished_at` in indicium, and are mapped; neither is
  renamed. The differences are listed under *Examples*.

ASTRA never points at an indicium record. A review verdict, a selection, a
mandate and an observation stay out of ASTRA, because each is a judgement by
someone about something, which indicium already models with attribution,
digests of what was judged and history.

### 5. Privacy

A run record holds no prompt, no model or tool payload, no text written by a
person, and no file content: references and hashes only
(`CONTENT_IN_RUN_RECORD`). Paths are relative. Per-event detail stays in the
external `event_log`, which the record names by reference and hash and never
embeds; whoever keeps that log decides who may read it.

Content hashes are unsalted SHA-256, as everywhere in ASTRA, so that anyone
holding the same bytes gets the same hash. That is what makes them checkable,
and it has two costs:

- **Confirmation by guessing.** For short or predictable content (a prompt
  template, a one-line configuration, a short text, a well-known file) anyone
  who can guess candidates can confirm which one was used, by hashing each.
  A hash hides nothing that can be guessed.
- **Linking.** The same hash in two records, or in a record and elsewhere,
  shows the same bytes were used in both.

A writer that must not reveal such content leaves out its `content_hash` and
keeps the reference. ASTRA defines no keyed digest: a `content_hash` that
isn't the SHA-256 of the bytes would break every check that relies on it.

Actor ids, labels and timestamps can identify people and their working hours.
Writers SHOULD use pseudonymous actor ids for people when records are shared
beyond the people concerned. An actor's `same_as` identifies a person outright
(an ORCID IRI does), whatever the actor id: writers leave it out of records
shared beyond the people concerned unless that person agreed.

## Examples

### Conformance example

`tests/data/valid/RunRecord-001-agent-run.json`: a coding agent's run for a
member, under the universe `baseline` with one decision changed. It called two
models, the second through a subscription whose price is unknown, so that
model's entry in `usage_by_model` has no `cost`, and neither has the run.
Records are JSON (`INVALID_SYNTAX`), and a JSON text is also a YAML 1.2
document.

```json
{
  "spec_version": "0.2.0",
  "id": "run_5d2a9c71e04b8f36",
  "commit": "4015b57a143aec5156fd1444a017a32137a3fd0f",
  "universe": "baseline",
  "decisions": {"model": "model_x"},
  "actors": {
    "member_a": {"kind": "human"},
    "agent:coder": {"kind": "agent", "label": "Coding agent"},
    "tool:harness": {"kind": "tool", "version": "1.4.2", "same_as": "https://example.org/harness/releases/1.4.2"}
  },
  "executor": "agent:coder",
  "on_behalf_of": "member_a",
  "started": "2026-11-25T14:00:00.000Z",
  "finished": "2026-11-25T15:12:41.503Z",
  "execution": "executed",
  "exit_code": 0,
  "stop_reason": "completed",
  "models": ["example-provider/model-x", "example-provider/model-y"],
  "model_calls": 57,
  "tokens": {
    "input": 1204511,
    "output": 38120,
    "reasoning": 9311,
    "cache_read": 1011200,
    "cache_write": 64002
  },
  "usage_by_model": [
    {
      "model": "example-provider/model-x",
      "model_calls": 51,
      "tokens": {"input": 1120400, "output": 35020, "reasoning": 9311, "cache_read": 1011200, "cache_write": 64002},
      "cost": {
        "amount": 4.82,
        "currency": "EUR",
        "basis": "estimated",
        "price_list": {"ref": "prices/example-provider-2026-11-01.json", "content_hash": "sha256:0b9e5c2f1d7a4e3b8c6f9a2d5e1b7c4f8a3d6e9b2c5f8a1d4e7b0c3f6a9d2e5b"}
      }
    },
    {
      "model": "example-provider/model-y",
      "model_calls": 6,
      "tokens": {"input": 84111, "output": 3100, "reasoning": 0, "cache_read": 0, "cache_write": 0}
    }
  ],
  "code": [
    {"ref": "tool:harness@1.4.2", "content_hash": "sha256:e5da4a1cdb3c241cc8b3f2a9d7ba70a679960729bd9d8700791d412b34feef97"}
  ],
  "configuration": [
    {"ref": "config/agent.toml", "content_hash": "sha256:776202cb9fdc3914b95dab8a3596f8810b887bed6e570c610bd42013ce04074b", "size": 812},
    {"ref": "prompts/system.md", "content_hash": "sha256:3fc92f2e9bcf983be7b820177eeb86b9c6044b28e9dd660737d171a9e14932e8", "size": 5120}
  ],
  "environment": [
    {"ref": "urn:example:env:sha256:5564fb144d12fed34a4208836774c51c894f91a889dae40fa03222c3b91ed68f"}
  ],
  "inputs": [
    {"ref": "cards/in_force.json", "content_hash": "sha256:61a211c78503fe616be2f29106c6b62ec4e0ace7176f677e1afe748ddcdabf72", "size": 20431}
  ],
  "outputs": [
    {"ref": "output1#baseline", "content_hash": "sha256:8cfff2eadfc1898477ff96f47e090f005649910ddf85488034b4575e9d97a8cc", "size": 4812331}
  ],
  "data_origin": "real",
  "event_log": {
    "ref": "logs/run_5d2a9c71e04b8f36.events.jsonl",
    "content_hash": "sha256:1e5ba78423717724298e36a9a58ce836ce82f74827368c40449689e7654cc69e",
    "size": 88213
  }
}
```

Invalid fixtures, each wrong for one reason:

- `RunRecord-201-no-executor.json` (`MISSING_FIELD`)
- `RunRecord-202-null-cost.json`, `"cost": null` (`WRONG_TYPE`)
- `RunRecord-203-executed-with-exit-code-1.json` (`EXIT_CODE_MISMATCH`)
- `RunRecord-204-currency-lowercase.json`, `"currency": "eur"` (`INVALID_VALUE`)
- `RunRecord-205-tokens-without-output.json` (`MISSING_FIELD`)
- `RunRecord-206-actor-kind-robot.json` (`INVALID_VALUE`)
- `RunRecord-207-absolute-ref.json`, `"ref": "/home/a/data.csv"` (`INVALID_PATH`)
- `RunRecord-208-time-with-offset.json`, `"started": "2026-11-25T14:00:00+01:00"` (`INVALID_VALUE`)
- `RunRecord-209-usage-model-not-in-models.json` (`UNKNOWN_MODEL`)
- `RunRecord-210-usage-calls-do-not-sum.json`, entries of 51 and 5 calls for `"model_calls": 57` (`USAGE_SUM_MISMATCH`)
- `RunRecord-211-same-as-not-an-iri.json`, `"same_as": "0000-0002-1825-0097"` (`INVALID_VALUE`)

### Mapping: ASB's tags to run records

ASB writes one run record for the build, `.astra/runs/<build_id>.json`, and one
per stage, `<build_id>__<stage>.json`, with `part_of: <build_id>` (ids slugged
to the id grammar). The configuration stays where it is: one decision per
model and per analytical flag in the `astra.yaml`, selected by the record's
`decisions`. ASB keeps writing its tags during a deprecation window (below).

| ASB tag (where) | Run record | Note |
|---|---|---|
| `asb:build_id` (root) | `id` | slugged |
| `asb:status` (stage) | `execution` | `complete` → `executed`; other values (unverified) → `failed` or `partial` |
| `asb:pipeline_health` (root) | `execution` | `executed` unless the build stopped; a degraded but finished build stays `executed` and keeps the tag |
| `asb:started_at`, `asb:finished_at` (stage) | `started`, `finished` | converted to UTC with milliseconds |
| `asb:duration_seconds` (root) | `finished` − `started` | not stored; the tag stays when only the duration is known |
| `asb:calls` (root, stage) | `model_calls` | |
| `asb:llm_errors` (root, stage) | none | stays a tag; failed calls are counted in `model_calls` |
| `asb:error_rate` (root) | none | derived; stays a tag |
| `asb:tokens_in`, `asb:tokens_out` | `tokens.input`, `tokens.output` | left out when the manifest has none, never `0` (`UNKNOWN_AS_ZERO`) |
| `asb:cost_usd` | `cost: {amount, currency: USD, basis}` | ASB sets `basis` from where its cost comes from (unverified: provider-reported or computed); left out for calls through a subscription harness, which ASB records as `0.0` today; an estimate names its price list in `price_list` |
| `asb:package_version` | `actors["tool:asb"].version` | ASB is the `tool` actor |
| `asb:git_sha`, `asb:git_dirty` | `commit` | only a full commit id, and only when `git_dirty` is false; else left out, and the tags stay |
| `asb:deps_hash` | `environment[]` | `ref: urn:asb:deps:<hash>`; a `content_hash` only if the hash is the SHA-256 of a lock file's bytes (unverified) |
| `asb:prompt_version` (model decision) | `configuration[]` | `ref: prompts/<agent>@<version>`, with a `content_hash` of the prompt file if ASB has it |
| `asb:seed`, `asb:temperature` (model decision) | decisions, as today | configuration, not run facts |
| stage `input_sha256`, `output_path` | `inputs[]`, `outputs[]` | `content_hash: sha256:<hex>` if it is the SHA-256 of the bytes (unverified) |
| per-agent model (decision `model_<agent>`) | `decisions` and `models` | the decision stays; `models` lists what was called, and `usage_by_model` may break a stage's usage down by model |
| `asb:cards_emitted`, `asb:figure_parser`, `asb:parser_degraded` | none | build outputs and health, not run facts; stay tags |
| `asb:build_journey` and the `build_journey` sub-analysis | none | the journey is ASB's own record; `trace/build_trace.yaml` may be named in `outputs` |

The run records' readers today are ASB's renderer and run viewer
(`astra_render/render.py`, `observatory/views/journey.py`) and its ARA export
(`ara_export/layers.py`), which read `run_trace.yaml`.

### Mapping: indicium's `execution_receipt` to run records

| indicium (trace profile v2) | Run record | Difference |
|---|---|---|
| `executor` | `executor` | same; both must be a declared actor |
| `actors[].kind` `human`/`agent`/`tool`/`unknown` | `actors.<id>.kind` | same values; ASTRA keys actors by id, per record |
| source `role: assistant` | `kind: agent` | indicium's role name for an agent's source |
| `code`, `configuration`, `environment` | same names | indicium requires each non-empty; ASTRA makes them optional, so a run record without them yields no receipt |
| `inputs`, `outputs` | same names | indicium requires them to equal the activity's `lineage`; ASTRA has no lineage |
| reference `{source_id, digest}` | `{ref, content_hash, size}` | indicium's `digest` is bare hex of the canonical JSON of the source's metadata (which holds the bytes' `sha256`); ASTRA's `content_hash` is `sha256:` and the hex of the bytes |
| `started_at`, `finished_at` | `started`, `finished` | name: each side keeps its own and neither is renamed, `started_at` ↔ `started` and `finished_at` ↔ `finished` (ASTRA's are the names RFC 0005 gave `Generation`); form: indicium takes any ISO 8601 time with an offset, ASTRA only UTC with milliseconds and `Z` |
| `exit_code` (required integer) | `exit_code` (optional) | ASTRA leaves it out where there was no process; indicium needs one |
| `model_calls` (integer or `null`) | `model_calls` (left out when unknown) | `null` ↔ absent |
| `tokens: {input, output}` exactly, or `null` | `tokens` with `input`, `output` and optional `reasoning`, `cache_read`, `cache_write` | the extra keys are dropped for indicium; totals are kept because `input` includes cached tokens |
| `cost: {amount, currency, basis}` or `null` | same keys, left out when unknown | indicium's `currency` is any non-empty string; ASTRA's is ISO 4217 |
| `basis: measured`/`estimated` | same | |
| activity `trace.execution` `executed`/`failed`/`partial` | `execution` | indicium also has `planned` and `not_applicable`, which a run record never needs |
| activity `trace.data_origin` incl. `unknown` | `data_origin`, left out when unknown | `unknown` ↔ absent |
| activity record `id` | `id` | an ASTRA run id is a valid indicium id |
| none | `part_of`, `on_behalf_of`, `commit`, `universe`, `decisions`, `models`, `usage_by_model`, `stop_reason`, `event_log`, `cost.price_list`, actor `label`, `version`, `same_as` | ASTRA only; an exporter can name the run record as a source to carry them |
| `mandate`, `selection`, `review`, `trace.method`/`scope`/`coverage` | none | indicium only, by design (section 4) |

Same exit-code rule on both sides: `executed` needs `0`, `failed` needs a
non-zero code; for `partial` nothing is said.

### Mapping: a lab tool's agent run and judge call

The first lab tool to use this (Collegium) keeps its own store and exports run
records on request. Its field design names these fields; the mapping below is
by field, not by storage.

Agent run (one per member's agent session):

| Field | Run record |
|---|---|
| run id | `id` |
| executor (the agent) | `executor`, an actor of kind `agent` |
| the member who launched it | `on_behalf_of`, an actor of kind `human` (simulated members: kind `agent`) |
| harness name and pin | `code[]` and the harness as a `tool` actor with `version` |
| model, route | `decisions` (the configuration is an ASTRA analysis) and `models` |
| cards in force | `inputs[]` with `content_hash` |
| started, ended | `started`, `finished` |
| agent exit code | `exit_code` |
| state | `execution` |
| stopped because | `stop_reason` |
| model calls | `model_calls` |
| tokens input, output, reasoning, cache read, cache write | `tokens`; the harness's `input` may exclude cached tokens (unverified), in which case `input = input + cache_read + cache_write` |
| cost | `cost`, left out when unknown |
| data origin | `data_origin` (simulated-lab runs: `synthetic`) |
| harness session ids, tool-call log | `event_log` (the log, by reference and hash); tool-call counts stay in the log |

Judge call (one per call, failed and timed-out calls included):

| Field | Run record |
|---|---|
| receipt id | `id` |
| executor (the judge) | `executor`, an actor of kind `agent` |
| judge model | `models`, and a decision if it is configuration |
| judge prompt version | `configuration[]`: `ref` to the pinned prompt, `content_hash` when the prompt is known, else no hash |
| started, finished | `started`, `finished` |
| outcome `finding` | `execution: executed` |
| outcome `failed` | `execution: failed` |
| outcome `timeout` | `execution: failed`, `stop_reason: timeout` |
| outcome `window-refused` | `execution: failed`, `stop_reason: refused` |
| model calls | `model_calls` |
| tokens input, output | `tokens.input`, `tokens.output`, never `0` for unknown |
| cost amount, currency, basis | `cost`, left out for the pilot |
| (no exit code) | no `exit_code`: none has to be invented |
| candidate and finding ids, verdict, quotes, target digest | none: the review belongs to indicium, which binds it to the run by `id` |

## Implementation implications & migration

*Compatibility:* additive. Nothing changes in `astra.yaml`. A new record
schema sits beside the others, and `Generation` gains two optional fields,
`run` and `run_content_hash`. A 0.1 checker reads a provenance record of a newer minor version
through its open schema (`INVALID_RECORD`), so `generated_by.run` breaks no
check, and it never looks in `.astra/runs/`.

*Version:* ASTRA publishes one schema per minor version, shared by every patch
of it (*Schema files* in the specification), so a new record schema can't
arrive in a patch. This RFC targets **0.2.0**. If it is accepted before 0.1.0
leaves release candidates, the core team may prefer to fold it into 0.1.0; it
removes nothing that 0.1.0rc1 has.

*`astra-spec`:*

- `src/astra/schema/run.yaml` (new), `types.yaml` (`actor_id`,
  `currency_code`), `provenance.yaml` (`Generation.run`,
  `Generation.run_content_hash`).
- Generated: `run.schema.json`, `run.open.schema.json`, `run.context.jsonld`;
  `rules.json` gains the new codes.
- `docs/specification.md`: a *Runs* paragraph beside *record*, the rule rows of
  section 3 and the schema-files rows; `docs/llms.txt`: the run record in the
  concept list and rules; `docs/cli.md` if `astra check` reads runs.
- Tests: `RunRecord` fixtures under `tests/data/` and `"RunRecord": "run"` in
  `_SCHEMA_NAMES` of `tests/conftest.py`.

*`astra-tools`:* the checker reads `.astra/runs/`, and run records kept
elsewhere when it is told where, and applies the check rows; the runner MAY
write a run record per invocation and set `generated_by.run` and
`generated_by.run_content_hash`.

*Writers:*

- **ASB** moves `run_trace.yaml` to ASTRA 0.1 (it targets 0.0.12 today), keeps
  configuration as decisions, and writes run records as mapped above. It keeps
  its `asb:` tags for one ASTRA minor version after the one that carries this
  RFC (through 0.2.x if this lands in 0.2.0), then drops the tags that have a
  field. The window is ASB's own, since the tags are its convention.
- **indicium** derives `execution_receipt` from a run record by the table
  above, and cites the record as an `artifact` source.
- **A lab tool** exports run records on request from what it stores, with no
  change to how it stores them.

*Migration steps:* none for existing analyses. Existing runs have no record,
and none is required.

## Questions or objections

### Alternatives considered

- **ASTRA describes configuration only; indicium's trace profile carries the
  rest.** This is ASB's current split and the one a lab tool's field design
  first chose. It keeps ASTRA smallest. It was not chosen because the profile
  is unreleased, carries no version of its own and is Python checks rather
  than a schema; because its bundle freezes source bytes under a package
  digest, which suits a research memory but not a record committed beside an
  analysis; and because every ASTRA user who runs an agent would then need a
  second format to say who ran it and what it cost, as ASB's free tags show.
  RFC 0005 already put the runner, the environment and the times of a run into
  ASTRA; this finishes that thought for runs that are not one recipe. The
  alignment with indicium is kept by shared names and one id (section 4).
- **W3C PROV-O or the RO-Crate Workflow Run profiles as the record.** A run
  record already is PROV-O through its JSON-LD context (`prov:Activity`,
  `prov:wasAssociatedWith`, `prov:used`, `prov:generated`,
  `prov:startedAtTime`, `prov:endedAtTime`, `prov:wasInformedBy`). The RO-Crate
  Process, Workflow and Provenance Run Crate profiles describe an execution as
  a `CreateAction` with `agent`, `instrument`, `object`, `result`,
  `startTime`, `endTime` and `actionStatus`; ASB already writes a Workflow Run
  Crate per capsule (its docstring says so; unverified). Neither has tokens,
  model calls or cost, and adopting RO-Crate as ASTRA's run format would bring
  a second packaging model into the spec. Instead the documentation should
  give the mapping: `executor` → `agent`, `code` → `instrument`, `inputs` →
  `object`, `outputs` → `result`, `started`/`finished` →
  `startTime`/`endTime`, `execution` → `actionStatus`
  (`CompletedActionStatus`, `FailedActionStatus`), so exporters can write a
  crate from a run record.
- **OpenTelemetry semantic conventions for generative AI.** They describe
  model and agent operations as spans with `gen_ai.*` attributes (model
  requested and returned, input and output tokens, finish reasons, and in
  recent versions cached and reasoning tokens; exact names and their status
  not checked here). They are per call, streamed to a collector, and still
  marked as in development. A run record is one summary per run, kept with
  the analysis. They fit together: an OTel export is a good `event_log`, and
  a writer can fill `tokens`, `models`, `model_calls` and `stop_reason` by
  summing spans. Before acceptance, the token definitions here should be
  checked against the current conventions, and the names aligned in the docs.
- **Extend `Generation` instead of adding a record.** `Generation` is one per
  result. A judge call makes no result and an agent session may make many, so
  the run needs its own record; `Generation.run` links the two.
- **Keep free tags as a convention.** No types, no units, no difference
  between unknown and zero, nothing a checker can read.

### Drawbacks

- ASTRA "stays out of execution". A run record doesn't run anything, as a
  provenance record doesn't, but it does bring execution facts (cost, tokens,
  models) into the spec, and with them requests for more such fields.
- One more schema, five classes and about fifteen rules to maintain.
- Two names (`started`/`finished` and `started_at`/`finished_at`) mean the
  same thing in two formats used together; the mapping states the
  correspondence, and an exporter has to apply it.
- Costs invite comparisons across providers and price lists that `basis` and
  `price_list` alone don't make fair.
- Unsalted hashes can confirm guessable content (section 5).
- A busy tool can make many run records. It may keep them outside git, but
  then every record a result cites has to stay reachable wherever it is kept,
  and that store has to be kept alongside the repository.

### Resolved in drafting

1. **Time field names: both kept, and mapped.** Run records use `started` and
   `finished`, the names RFC 0005 already released in `Generation`; indicium
   keeps `started_at` and `finished_at`. Nothing is renamed in either format,
   and the indicium mapping table states the correspondence.
2. **`commit` stays optional, and a record without it may set `universe` and
   `decisions`.** They are then read at the commit that adds the record
   (`UNKNOWN_RUN_UNIVERSE`, `UNKNOWN_OPTION`). The absence of `commit` says
   that the run came from a working tree with changes or was imported, which
   is worth recording rather than refusing.
3. **Per-model usage: an optional `usage_by_model`.** Several models used
   within one run are a breakdown of that run's usage, not separate runs, so
   a record lists them in entries summing to its totals (`UNKNOWN_MODEL`,
   `USAGE_SUM_MISMATCH`). A child run with `part_of` remains the way to record
   work that is a run of its own.
4. **Run records may be kept outside the analysis's repository.** The format
   and the rules apply wherever a record is kept, and a committed record stays
   at `.astra/runs/{id}.json`. A record that another record cites must be
   reachable by its reader (`MISSING_RUN`) and should be cited with the
   SHA-256 of its bytes, which a provenance record carries in the one field
   this adds to RFC 0005's record, `generated_by.run_content_hash`
   (`RUN_HASH_MISMATCH`).
5. **An `estimated` cost should name its price list** in `cost.price_list`, so
   that two estimates can be compared. Without it the record stays valid, with
   a warning (`MISSING_PRICE_LIST`).
6. **Actor ids stay local; an actor may carry `same_as`.** The id grammar is
   unchanged, so ids stay short and match indicium's. An optional absolute
   IRI, such as an ORCID IRI or a tool release's URL, links the same actor
   across analyses; since it identifies a person, writers leave it out of
   records shared beyond the people concerned unless that person agreed
   (section 5).

### Unresolved questions

None remain from the first draft; the six it raised are answered above.

Signing run records (in-toto, SLSA) stays with RFC 0005's future *Signed
records*. `on_behalf_of` records who a run was for, not an authorisation;
authorisation is indicium's `mandate`.

## References

- [RFC 0005](0005-result-provenance-and-layout.md): provenance records,
  `Generation`, `.astra/`; [RFC 0008](0008-settling-the-schema-for-v0.1.md):
  `spec_version`, maps keyed by id; [RFC 0003](0003-cross-universe-artifact-references.md)
  and [RFC 0004](0004-composition.md): universes and addresses.
- ASB: `src/agentic_science_builder/astra_export/run_trace.py` and
  `tests/test_astra_run_trace.py` (AgenticScienceBuilder `main` 25e76097);
  `llm_logging.py` and `build_manifest.py` for how counts and costs are taken.
- indicium trace profile v2, branch `codex/trace-recommendations` (42b6b17,
  unreleased): `docs/adapters/research-memory-trace-profile.md`,
  `docs/adapters/research-memory-assurance.md` (*Execution receipts*),
  `src/indicium/research_memory/{bundle,trace_profile,execution_receipts,research_lineage,assurance}.py`.
- Collegium field design and its addendum (2026-11-18), which list what an
  agent run, a judge call and a curator's decision need recorded.
- [W3C PROV-O](https://www.w3.org/TR/prov-o/).
- [Workflow Run RO-Crate](https://www.researchobject.org/workflow-run-crate/)
  (Process, Workflow and Provenance Run Crate profiles).
- [OpenTelemetry semantic conventions for generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/).
- [ISO 4217](https://www.iso.org/iso-4217-currency-codes.html) currency codes.
