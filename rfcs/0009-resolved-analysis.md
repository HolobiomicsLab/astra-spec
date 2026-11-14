---
rfc: 0009
title: The resolved analysis
status: Draft
authors:
  - Francois Lanusse (@eiffl)
created: 2026-11-14
tracking-issue:
superseded-by:
---

## Context

The [specification](../docs/specification.md) names three tools: a validator,
a runner and a checker. Each of them needs the analysis with every rule
applied: which results it plans, where each one goes, what each one reads and
which option each universe selects. So does a page that shows an analysis to a
reader.

Nothing in ASTRA says what that looks like, so each tool has its own shape for
it:

- astra-tools gives Python objects: the planned results, each holding the
  objects it reads, and a separate view of one universe. They aren't a
  document, and no other language can take them.
- The TypeScript SDK (`@astra-spec/sdk`) gives a `ResolvedAnalysisBundle`: one
  document per selected universe, a tree of analyses whose records each carry
  a `canonicalPath`, with the universe written into the records
  (`selectedOptionId` and `active` on a decision, `active` on an output), each
  output's `provenance.inputPaths` and `provenance.decisionPaths`, and a list
  of `bindings` from outputs to their files. It was written for ASTRA 0.0.x,
  with its universe files, `from` and `when`.

The two were written apart and nothing says they agree. A runner built on one
and a page built on the other can read the same `astra.yaml` and differ on
what it means, and a third tool has nothing to build on but the rules
themselves.

This RFC proposes one data model for an analysis with its rules applied, the
**resolved analysis**, and one for what a checker finds, the **check report**.
Both are plain data, defined once here, so that every tool in every language
gives and takes the same thing.

## Proposal

A resolved analysis is a document a tool makes from a valid analysis. It
holds three things:

- every analysis of the [stack](../docs/specification.md#rules), with what it
  declares for a person to read: its inputs, decisions and their options,
  outputs and insights;
- each analysis's universes, with the option each selects for every decision
  it uses;
- the planned results, each with its places, the command that makes it and
  what each entry of its output's `inputs` reads.

A runner reads the results. A page reads all of it. Neither applies a rule of
the specification: the tool that made the document already has.

Below, `?` marks a field that may be left out, `[T]` is a list and `{K: V}` a
map. A `ref` and an `id` are strings, described in section 6.

### Modifications

#### 1. The model

`ResolvedAnalysis`, the document:

| Field | Type | What it holds |
|---|---|---|
| `spec_version` | string | the version of ASTRA the document is written at, such as `"0.1.0"` |
| `analyses` | [`Analysis`] | the stack: the driving analysis first, then every analysis it declares, directly or not |
| `results` | [`Result`] | the planned results, each after every result it reads |
| `notes` | [`Diagnostic`] | what an author may want to tidy, none of which breaks a rule: an output with no planned result, say, or one a switch leaves unbuilt in a universe |

`Analysis`, one analysis of the stack:

| Field | Type | What it holds |
|---|---|---|
| `key` | string | its canonical address (section 6): `""` for the driving analysis, else the declarations that reach it, such as `stage` or `stage.inner` |
| `folder` | string | its folder, as a path from the driving analysis's folder: `.` for the driving analysis, `stage`, `../sibling`. For an analysis read at a commit, from the top of the repository it is read in |
| `commit` | `Commit`? | there when the analysis is read at a commit |
| `label`, `description` | string? | as its file writes them |
| `tags` | [string] | as its file writes them |
| `declares` | {id: key} | each of its declarations, and the `key` of the analysis it reaches |
| `inputs` | [`Input`] | |
| `decisions` | [`Decision`] | |
| `outputs` | [`Output`] | its outputs other than switches |
| `prior_insights`, `findings` | [`Insight`] | |
| `universes` | [`Universe`] | its own universes, imports aside |

`Commit` has an `id`, the full commit id, and a `repository?`, the URL of
another repository. `repository` is left out for the repository that holds the
driving analysis.

What an analysis declares:

| Type | Fields |
|---|---|
| `Input` | `ref`; `label?`, `description?`; `tags`; `path`, from the driving analysis's folder or, for an analysis read at a commit, in that commit's tree |
| `Decision` | `ref`; `label?`, `rationale?`; `tags`; `options`: [`Option`] |
| `Option` | `id`; `label?`, `description?`; `tags`; `excluded?`, the reason it is excluded; `insights`: [ref], the prior insights that support it |
| `Output` | `ref`; `label?`, `description?`; `tags`; `type`; `format`; `resources?`, with `cpus?`, `memory?`, `time_limit?`, `disk?` and `gpus?`; `decisions`: [ref], the decisions it is made with |
| `Insight` | `ref`; `kind`, `finding` or `prior_insight`; `label?`; `claim`; `description?`, `scope?`; `tags`; `evidence`: [`Evidence`] |
| `Evidence` | exactly one of `doi`, with `version?`, and `cites`: [id], the results it cites; `quote?`, with `exact`, `prefix?` and `suffix?`; `location?`, with `value` and `conforms_to?` |
| `Universe` | `ref`; `label?`, `description?`; `tags`; `selection`: {ref: option id}, the option it selects for each decision it uses |

What the driving analysis plans:

| Type | Fields |
|---|---|
| `Result` | `id`; `output`: ref; `universe?`: ref, left out for an output built once; `command?` (section 8); `inputs`: [`Source`], one for each entry of its output's `inputs`, in their order; `places`: [`Place`], one for each of its names |
| `Source` | `entry`, as the output's `inputs` writes it; and exactly one of `input`: ref, the input the entry reads, and `result`: id, the result it reads |
| `Place` | `address`; `record_path`; `bytes_path`. Both paths are from the driving analysis's folder |

A `Diagnostic` is what the validator already reports, with one more level: a
`code`; a `level`, `error`, `warning` or `note`; a `message`; and a
`location`, with `file`, a JSON `pointer` into it, and `line?`, `column?`,
`key?`, `commit?` and `url?`.

#### 2. Outcomes, not instructions

The model holds what the specification's rules give, not what they start from.
Where a file writes something only for a rule to act on, the model holds the
outcome and drops the instruction. What a file writes to be read by a person
stays as it is: labels, descriptions, rationales, claims, quotes.

| A file writes | The rule | The model holds |
|---|---|---|
| `requires`, `incompatible_with` | constraints on what a universe may select | nothing: only a valid analysis resolves, so every selection already meets them |
| `derived_from` | a universe inherits another's selection | `Universe.selection`, the effective one |
| a universe's keys for decisions it doesn't use | a universe uses the decisions its results are made with | nothing: `selection` holds exactly the decisions it uses |
| a switch and its `cases` | in each universe a switch is read as the case its decision selects | `Result.inputs`, which name what each entry reads there. Where a switch leaves an output unbuilt there is no result, and a note says why |
| an output's `inputs`, with addresses and pins | which input, or which result in which universe, an entry reads | `Result.inputs`, one `Source` for each entry |
| the recipe's `command`, a template | placeholders become paths and selected options | `Result.command`, rendered |
| evidence's `artifact` | the results it cites, in each universe where it names no pin | `Evidence.cites` |
| `analyses`, with paths, URLs and commits | which analysis each declaration reaches | `Analysis.declares`, `folder` and `commit` |
| an import of another analysis's universe | a pin names the universe it imports | the `universe` of the result that reads through it |

An option's `excluded` looks like a rule and stays: its reason is text a page
shows beside the option.

A page loses three things by this: switches, the constraints between options,
and which universe another was derived from. They are still in the file, for
a tool that wants the analysis as written.

#### 3. Nothing twice

A fact is in the model once. A derived fact is in the model only when working
it out takes a rule of the specification. Anything a reader gets by following
a reference is left out.

A result therefore doesn't carry:

| Not on a result | Where it is |
|---|---|
| the options it is made with | its output's `decisions`, looked up in its universe's `selection` |
| the folder its recipe runs in | the `folder` of its output's analysis |
| what its recipe asks for | its output's `resources` |
| whether it is a result from a commit, or runs at one | the `commit` of the analysis it belongs to, and of its output's analysis |
| the path of what an entry reads | the `path` of the input its `Source` names, or the `bytes_path` of the result it names |

It does carry its `command`, its `inputs` and its `places`, as each takes a
rule: the placeholders, the case each switch takes and each pin, and the
layout of results.

The model holds each link in the direction the file writes it. A reader that
wants the reverse, such as the outputs made with a decision, builds that index
itself.

#### 4. Only a valid analysis resolves

A tool gives a resolved analysis only of an analysis that is
[valid](../docs/specification.md#rules), and the validator's report otherwise.
Every field is then always there: no universe lacks an option for a decision
it uses, and no reference names nothing.

#### 5. Output, result, place

- An **output** is a declaration: one entry of `outputs`, with its type, its
  format and the decisions it is made with.
- A **result** is that output in one universe, or once for an output built
  once. It is what a runner builds and what a record describes.
- A **place** is where a result is kept: its address, the path of its record
  and the path of its bytes. A result has one place unless its analysis is
  driven through two chains of declarations
  ([`RESULT_IN_TWO_PLACES`](../docs/specification.md#rules)), and that is the
  only reason `Place` is a type of its own.

Universes belong to their analysis. A result that a pin reads in another
analysis's own universe names that universe, so the universes of every
analysis of the stack are in the model, each under its `Analysis`.

A universe's `selection` holds exactly the decisions it uses. It lacks none of
them (`MISSING_DECISION`). An option it inherits for a decision it doesn't use
changes nothing it builds, so it is left out; its own keys can't set a
decision it uses on no branch (`UNUSED_DECISION`) and shouldn't set one it
doesn't use (`INACTIVE_DECISION`). Whether a universe uses a decision is then
whether the decision is in its selection.

#### 6. References are ids

One element never holds another: it names it. A `ref` names an element among
all the analyses of the stack. It is the address an author would write, with
the collection made explicit:

```text
ref = [ <analysis key> "." ] <collection> "." <id>
```

The collections are the keys of the file: `inputs`, `outputs`, `decisions`,
`universes`, `prior_insights` and `findings`. The last two segments are always
the collection and the id, whatever a declaration is called, and what comes
before them is the `key` of the element's analysis.

| Names | Ref or id |
|---|---|
| an output of the driving analysis | `outputs.fit` |
| a decision of a declared analysis | `stage.decisions.method` |
| an input two declarations down | `stage.inner.inputs.catalog` |
| an element of an analysis read at a commit | `previous.outputs.fit` |
| an option: the decision's ref, then the option's id | `decisions.method.mcmc` |
| a piece of evidence: the insight's ref, then its index from zero | `findings.detection.0` |
| a result of the driving analysis: its address | `fit#baseline` |
| a result that belongs to another analysis: that analysis's key, `:`, its address | `stage:fit#v1` |

An analysis read at a commit needs no commit in its refs: its declaration
carries the commit, so `previous.outputs.fit` is already another element than
`outputs.fit`.

One analysis can be reached through several chains of declarations, so the
model picks one, its **canonical address**: the shortest chain that reaches it
and, among equals, the first in alphabetical order. The rule rests on nothing
but the files, so two tools agree on it. An author's own references stay free:
any address that reaches an element names it, and resolves to the same ref.

A result's id takes a colon after its owner because a dot could name two
results: `stage.fit#v1` is the driving analysis's own result of a driven
`stage` in its universe `v1`, and would also be `stage`'s own result in its
own `v1`.

The alternative is to name an analysis by its identity, its folder and
commit. That never changes when a declaration is added, where a canonical
address can, but it puts a full commit id in every ref. The identity stays in
the model as `Analysis.folder` and `Analysis.commit`.

#### 7. One frame for paths

Every path in the model is from the driving analysis's folder, the folder the
tool was given: an analysis's `folder`, an input's `path`, a place's
`record_path` and `bytes_path`, and the `file` of a diagnostic. A file read at
a commit isn't in the working tree, so it is named in that commit's tree, from
the top of its repository, and its analysis's `commit` says which.

#### 8. The command

`Result.command` is the recipe's command with its placeholders filled in:
each path relative to the folder the recipe runs in, the folder of its
output's analysis
([`WRONG_WORKING_DIRECTORY`](../docs/specification.md#rules)). The
specification fixes that folder, where every file is and which options are
selected, so there is one rendering and the tool that resolves gives it.

`command` is left out for a result that isn't built from the working tree
alone: a result from a commit, one whose recipe runs on an analysis's files at
a commit, and one that reads a file at a commit. Such a file has no place in
the working tree, so no rule says what path the command would name.

#### 9. Plain data

- The model has no methods. What a tool does with it is that tool's.
- It holds only what JSON holds: strings, numbers, lists, and maps with
  string keys.
- A field with no value is left out, never null, as in an `astra.yaml`. A
  list or a map is always there, and empty where it holds nothing.
- A set is written as a sorted list.
- Field names are in snake_case in every language, as in the files.

#### 10. What stays outside

The model is a function of the files of the stack. Three things change while
those files stay as they are, so they aren't in it:

- how each result stands, which is the check report's (section 11);
- the bytes of an input or a result, which a tool reads at the path the model
  gives;
- the works evidence cites by `doi`: their text, titles and authors.

#### 11. The check report

A checker gives one document for a resolved analysis:

| Type | Fields |
|---|---|
| `CheckReport` | `analysis`: `ResolvedAnalysis`, the one it checked; `statuses`: {id: `Status`}, one for each planned result; `evidence`: [`EvidenceStatus`] |
| `Status` | `state`, `unchanged`, `stale` or `missing`; `bytes?`, `present` or `absent`; `record?`, the result's record as the [record schema](../docs/specification.md#schema-files) has it; `why`: [`Diagnostic`]; `stale_inputs`: [entry] |
| `EvidenceStatus` | `insight`: ref; `index`, from zero; `state`; `why`: [`Diagnostic`] |

There is one status for each result. A result with two names has a record at
each place, and its status is the worse of the two, with `why` saying which
place.

A status keeps apart the two ways a result is stale:

- `why` holds what is wrong with the result itself: it has no valid record,
  or its output's declaration, an option it is made with or the content of an
  input has changed since its record's commit.
- `stale_inputs` names the entries of its output's `inputs` whose result is
  stale or missing.

A result that is stale only through `stale_inputs` is unchanged again once
those results are rebuilt with the hashes its record gives. Whether to wait
for them and skip the rebuild is a runner's to decide, not the checker's. The
split, with the hash of each input in the record, is what a runner needs to
decide it.

## Examples

### Example 1: a resolved analysis

The analysis has one input, one decision, an output built in two universes
and an output built once that reads both:

```yaml
spec_version: "0.1.0"
label: Example analysis
inputs:
  input1: {path: data/input1.csv}
decisions:
  decision1:
    rationale: Both are reasonable, and they may give different results.
    options: {a: {}, b: {}}
outputs:
  output1:
    type: metric
    format: json
    inputs: [input1]
    decisions: [decision1]
    recipe:
      command: python src/output1.py {inputs.input1} {decisions.decision1} {output}
  output2:
    type: table
    format: csv
    inputs: [output1#u1, output1#u2]
    recipe:
      command: python src/output2.py {inputs} {output}
universes:
  u1: {decisions: {decision1: a}}
  u2: {decisions: {decision1: b}}
findings:
  insight1:
    claim: Options a and b of decision1 give different results.
    evidence:
      - artifact: output2
```

Its resolved analysis, written by hand for this RFC until a tool gives it:

```json
{
  "spec_version": "0.1.0",
  "analyses": [
    {
      "key": "",
      "folder": ".",
      "label": "Example analysis",
      "tags": [],
      "declares": {},
      "inputs": [
        {"ref": "inputs.input1", "tags": [], "path": "data/input1.csv"}
      ],
      "decisions": [
        {
          "ref": "decisions.decision1",
          "rationale": "Both are reasonable, and they may give different results.",
          "tags": [],
          "options": [
            {"id": "a", "tags": [], "insights": []},
            {"id": "b", "tags": [], "insights": []}
          ]
        }
      ],
      "outputs": [
        {
          "ref": "outputs.output1",
          "tags": [],
          "type": "metric",
          "format": "json",
          "decisions": ["decisions.decision1"]
        },
        {
          "ref": "outputs.output2",
          "tags": [],
          "type": "table",
          "format": "csv",
          "decisions": []
        }
      ],
      "prior_insights": [],
      "findings": [
        {
          "ref": "findings.insight1",
          "kind": "finding",
          "claim": "Options a and b of decision1 give different results.",
          "tags": [],
          "evidence": [{"cites": ["output2"]}]
        }
      ],
      "universes": [
        {
          "ref": "universes.u1",
          "tags": [],
          "selection": {"decisions.decision1": "a"}
        },
        {
          "ref": "universes.u2",
          "tags": [],
          "selection": {"decisions.decision1": "b"}
        }
      ]
    }
  ],
  "results": [
    {
      "id": "output1#u1",
      "output": "outputs.output1",
      "universe": "universes.u1",
      "command": "python src/output1.py data/input1.csv a results/u1/output1.json",
      "inputs": [{"entry": "input1", "input": "inputs.input1"}],
      "places": [
        {
          "address": "output1#u1",
          "record_path": ".astra/records/u1/output1.json",
          "bytes_path": "results/u1/output1.json"
        }
      ]
    },
    {
      "id": "output1#u2",
      "output": "outputs.output1",
      "universe": "universes.u2",
      "command": "python src/output1.py data/input1.csv b results/u2/output1.json",
      "inputs": [{"entry": "input1", "input": "inputs.input1"}],
      "places": [
        {
          "address": "output1#u2",
          "record_path": ".astra/records/u2/output1.json",
          "bytes_path": "results/u2/output1.json"
        }
      ]
    },
    {
      "id": "output2",
      "output": "outputs.output2",
      "command": "python src/output2.py results/u1/output1.json results/u2/output1.json results/output2.csv",
      "inputs": [
        {"entry": "output1#u1", "result": "output1#u1"},
        {"entry": "output1#u2", "result": "output1#u2"}
      ],
      "places": [
        {
          "address": "output2",
          "record_path": ".astra/records/output2.json",
          "bytes_path": "results/output2.csv"
        }
      ]
    }
  ],
  "notes": []
}
```

- `output1` has two results and `output2` one, with no `universe`
- `u1` selects `a`: `output1#u1` is made with it, which its command already
  shows
- `insight1` cites `output2`'s one result
- nothing of `inputs: [output1#u1, output1#u2]` or of the two templates is
  left: the results say what each entry reads and what to run

### Example 2: a status

`output1`'s command is then edited and committed. A check finds `output1#u1`
stale in itself, and `output2` stale only through what it reads. Two of the
report's statuses, again written by hand, with hashes and commit ids left
as placeholders:

```json
{
  "output1#u1": {
    "state": "stale",
    "bytes": "present",
    "record": {
      "spec_version": "0.1.0",
      "address": "output1#u1",
      "content_hash": "sha256:<hash>",
      "size": 17,
      "generated_by": {
        "commit": "<commit id>",
        "inputs": [
          {"entry": "input1", "content_hash": "sha256:<hash>", "size": 4}
        ],
        "environment": {"iri": "urn:example:environment1"},
        "runner": {"id": "example-runner", "version": "1.0.0"}
      }
    },
    "why": [
      {
        "code": "STALE_RESULT",
        "level": "error",
        "message": "its recipe's command isn't the one it had at <commit id>",
        "location": {"file": ".astra/records/u1/output1.json", "pointer": ""}
      }
    ],
    "stale_inputs": []
  },
  "output2": {
    "state": "stale",
    "bytes": "present",
    "record": {
      "spec_version": "0.1.0",
      "address": "output2",
      "content_hash": "sha256:<hash>",
      "size": 42,
      "generated_by": {
        "commit": "<commit id>",
        "inputs": [
          {"entry": "output1#u1", "content_hash": "sha256:<hash>", "size": 17},
          {"entry": "output1#u2", "content_hash": "sha256:<hash>", "size": 17}
        ],
        "environment": {"iri": "urn:example:environment1"},
        "runner": {"id": "example-runner", "version": "1.0.0"}
      }
    },
    "why": [],
    "stale_inputs": ["output1#u1", "output1#u2"]
  }
}
```

`output2`'s `why` is empty. If rebuilding `output1` in both universes gives
the hashes `output2`'s record lists, `output2` is unchanged, and a runner can
leave it.

### Example 3: a runner

A runner in Python, on astra-tools' three functions. It resolves, checks
once, runs each command in its output's analysis's folder, and writes what
its run made:

```python
analysis = astra.resolve(root)        # a valid analysis, or the validator's report
report = astra.check(root, analysis)  # once, where git is
folder_of = {o.ref: a.folder for a in analysis.analyses for o in a.outputs}

for result in analysis.results:       # each after the results it reads
    status = report.statuses[result.id]
    ...                               # the runner's own plan: what to build, and where

# For one result, once the results it reads are built: the runner's own rule.
# `now` gives, by entry, the hash of each result it reads as it now is.
recorded = (
    {i.entry: i.content_hash for i in status.record.generated_by.inputs}
    if status.record
    else {}
)
if status.why or any(recorded.get(entry) != now[entry] for entry in now):
    run(result.command, cwd=root / folder_of[result.output])
    ...                               # hash what it made
    astra.write_records(root, result, content_hash, size, generated_by)
```

## Implementation implications & migration

*Compatibility:* additive. Nothing changes in what is a valid `astra.yaml` or
a valid record, and the specification's table of rules stays as it is.

*Schema:* the model becomes a third LinkML schema in `src/astra/schema/`,
beside `analysis.yaml` and `provenance.yaml`. The recipes the repository has
generate the rest from it, as they do for the other two: `just gen-schema`
its JSON Schema in `project/jsonschema/`, `just gen-python` the Python
datamodels in `src/astra/datamodel/`, `just gen-doc` its pages of the field
reference, and `gen-typescript`, which `just gen-project` runs, the
TypeScript types. `just check-generated` then covers the new files.

*Conformance:* each valid case of `conformance/analyses/` gains its expected
resolved analysis, which every tool that resolves must give for it.

*Tools:* astra-tools returns the model from `resolve` and the check report
from `check`
([astra-tools#138](https://github.com/astra-project/astra-tools/pull/138),
being rebuilt on it). The TypeScript SDK's `resolveAnalysis` changes from its
document of one universe to this model. A runner reads the results and the
statuses
([lightcone-cli#273](https://github.com/LightconeResearch/lightcone-cli/pull/273)
is the first).

*Versioning:* the schema files of a minor version are shared by every patch
of it, and a patch release never changes what is valid. This RFC adds a
schema and changes neither of the others nor `rules.json`, so it asks for no
new minor version: it can come in before 0.1.0 or in a patch after it. Once
released, the model is held as the other schemas are. A change that makes a
document written to it no longer hold waits for the next minor version, and
`tools/check_patch.py` compares its JSON Schema with the others.

*Migration steps:* none for an analysis or its records. A tool that reads the
TypeScript SDK's document of one universe moves to this model.

## Questions or objections

Settled while drafting:

- **Should a tool resolve what it can of an invalid analysis?** No. Only a
  valid analysis resolves (section 4), so no field is ever missing. A checker
  then gives the validator's report for an invalid analysis and no statuses.
  The conformance suite expects none for one.
- **One status for each place, or for each result?** For each result (section
  11). A runner builds a result whole, so it has no use for half of one.
- **Paths from the top of the repository, or from the driving analysis's
  folder?** From the folder (section 7). It is where a runner works and what a
  reader sees, and the validator's diagnostics name files the same way.
- **Are refs and result ids part of ASTRA?** They are a convention of this
  model, built from the specification's addresses. No author writes one, and
  no rule reads one.
- **Does a page get its model from a server?** That is a separate question,
  which this RFC leaves open. A TypeScript resolver stays, and this RFC only
  asks that it give the same model as any other.
- **Should the model answer whether a rebuild can be skipped?** No. An
  earlier draft had a function for it; that is a runner's logic. The check
  report gives a runner what it needs instead (section 11).
- **Should a result repeat what its output and its universe say?** No
  (section 3). A copy leaves a reader unable to tell which one holds.
- **Should switches, constraints and `derived_from` stay, for a page?** No
  (section 2). The model holds outcomes, and the file still holds the rest.

Open:

- **snake_case in TypeScript.** The SDK's present document mixes the files'
  snake_case with camelCase for what it adds (`canonicalPath`,
  `selectedOptionId`). One schema means one spelling.
- **Is the check report in the same schema as the resolved analysis?** A page
  shows statuses too, so both are to be defined once. Whether as one schema
  or two isn't settled.
- **Results from a commit.** A result that reads a file at a commit has no
  `command` (section 8), and a runner that fetches such a file would render
  the command from the template in the file. Whether the model should say
  where a fetched file goes, so that every result has a command, isn't
  settled. No runner builds from a commit today.
- **Evidence has no id.** A page that links to one piece of evidence names it
  by its insight and its index, which moves when evidence is added before
  it.
- **The order of lists.** For two tools to give the same document, every list
  needs an order: the analyses after the driving one, results that don't read
  one another, and which lists count as sets (section 9). The conformance
  cases will need it said.

## References

- [RFC 0003](0003-cross-universe-artifact-references.md): universes in
  `astra.yaml`, and pins
- [RFC 0004](0004-composition.md): declared analyses, addresses, driving, and
  where a result belongs
- [RFC 0005](0005-result-provenance-and-layout.md): records, places, and when
  a result is unchanged
- [RFC 0007](0007-analyses-in-other-repositories.md): analyses from a commit
- [The specification](../docs/specification.md): Elements, Results and Rules
- [The proposal this RFC was drafted from](https://gist-artifacts.dev/g/fc61a990c7bafa30b7a62af6d7ff9f74),
  with its diagrams
- [astra-tools#138](https://github.com/astra-project/astra-tools/pull/138) and
  [lightcone-cli#273](https://github.com/LightconeResearch/lightcone-cli/pull/273)
- [`@astra-spec/sdk`](https://github.com/LightconeResearch/astra-typescript),
  the TypeScript SDK, and its `ResolvedAnalysisBundle`
