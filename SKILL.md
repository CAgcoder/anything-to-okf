---
name: anything-to-okf
description: >-
  Convert files of ANY format — PDF, Word/.docx, PowerPoint/.pptx,
  Excel/.xlsx/.csv, HTML, Markdown, JSON/YAML, source code, plain text, images —
  into an Open Knowledge Format (OKF) bundle: a directory of markdown files with
  YAML frontmatter, cross-links, and index.md files, conformant to OKF v0.1. Use
  this skill whenever the user wants to turn documents, datasets, a folder of
  mixed files, a knowledge base, a spec, or an export into OKF, an "OKF bundle",
  a "knowledge bundle", or "Open Knowledge Format" — even if they just say
  "convert these to OKF" or "build a knowledge bundle from this folder".
---

# Anything → OKF

Turn arbitrary source files into a conformant **Open Knowledge Format (OKF)**
bundle: a directory of markdown files with YAML frontmatter, joined by markdown
cross-links and `index.md` listings — readable by humans, parseable by agents,
diffable in git. Full rules: [references/okf-spec.md](references/okf-spec.md);
read it before planning.

## How this skill works

**You (the agent) do the whole conversion directly.** You extract the content,
decide what the concepts are, name and group them, and write each concept's
markdown yourself — grounded in the extracted source. There is no CLI dispatch
and no LLM "merge pass": you are the LLM, so the reasoning that needs judgment
is yours to do inline.

Only two things are scripts, because they must be deterministic, not judgment:

- [scripts/generate_indexes.py](scripts/generate_indexes.py) — builds every
  `index.md` from concept frontmatter.
- [scripts/validate_bundle.py](scripts/validate_bundle.py) — checks OKF
  conformance (hard errors vs. soft warnings).

Both need PyYAML (`pip install pyyaml` if missing).

## Your role, and the voice of every concept

**Role (task context).** You are a knowledge engineer turning source material
into a trustworthy OKF bundle that humans and agents can rely on *without
re-reading the sources*. Your job is judgment — what the concepts are, how they
connect, what each one truthfully says. Parsing files is other skills' job; the
deterministic bookkeeping is the two scripts' job.

**Voice (tone) of a concept.** Each concept reads like a reference entry, not
marketing prose: factual, dense, neutral. Short declarative sentences,
definitions over adjectives, no hedging or filler ("this powerful table
enables…" is wrong). If a fact isn't in the source, stay silent — never smooth a
gap with a plausible sentence.

This skill supplies the durable half of the prompt — role, tone, background
([references/okf-spec.md](references/okf-spec.md) + the extracted source), the
rules below, and the worked example. The **runtime** supplies the other half: the
conversation so far and the user's actual request ("convert these to OKF"). Don't
hardcode those into anything.

## Workflow

### 1. Discover and classify inputs

List the input file(s)/directory. Classify each by extension. If given a URL or
an archive, fetch/extract first. Make a scratch dir for extracted text:
`mkdir -p <bundle>/.okf-work` — it holds working text and is **not** part of the
bundle (the scripts ignore non-`.md` files, but keep it out of the final tree).

### 2. Extract clean content per source — faithfully

Turn every source into faithful text/markdown. Method per format is in
[references/format-extraction.md](references/format-extraction.md) (office
formats use the bundled `pdf`/`docx`/`pptx`/`xlsx` skills; text/code/json read
directly; html via pandoc/WebFetch; images described visually). Write each
source's text to `<bundle>/.okf-work/<slug>.src.txt` if it helps you re-reason
later; small sources you can hold in context directly.

The cardinal rule is **fidelity**: preserve exact column names, types, enum
values, formulas, IDs, and numbers; keep tables as tables and code as code;
never invent to fill a gap. Everything downstream inherits this grounding (or
this hallucination), so get it right here.

### 3. Plan the concept set and the tree

Before writing anything, decide the concepts and the directory layout. This is
the step that makes the bundle coherent rather than a flat dump.

- **One concept = one unit of knowledge** — a table, dataset, API endpoint,
  metric, playbook, glossary term, source document, or code module. One rich
  source often yields several concepts; several near-identical sources collapse
  into one.
- **Canonicalize and dedupe.** If two sources describe the same thing under
  different names, make ONE concept and mention the aliases in the body. The id
  is just the file path minus `.md` (spec §2) — pick a stable, descriptive one;
  lowercase kebab-case is a sensible default, not a spec requirement.
- **Choose a directory structure** that mirrors how the knowledge groups
  (`tables/`, `datasets/`, `references/`, `playbooks/`, …). Layout is free; pick
  what aids navigation.
- **Assign each concept a `type`** — short, self-explanatory, reused across like
  concepts (`BigQuery Table`, `API Endpoint`, `Metric`, `Playbook`,
  `Glossary Term`, `Document`, …). The index groups by `type`, so be consistent.
- **Note the relationships** between concepts (FK, "uses", "derived from", …).
  You'll turn these into plain markdown links while writing — you already know
  them because you planned the set; no separate pass infers them.

Write the plan down (a short `concept-id → type → source` list) before writing.

### 4. Write each concept .md directly

Think through what this one source truthfully supports first (internally — don't
narrate the analysis), then write the file. Each
`<bundle>/<dir>/<concept-id>.md` follows this output contract:

1. **Start with the frontmatter block.** First line is `---`. `type` is the only
   hard requirement; add `title`, `description`, and `resource`/`tags`/`timestamp`
   when the source provides them. YAML safety: quote any value containing a
   colon, `#`, or quotes, or the parser will choke.
2. **Body in structured sections.** `# Schema` tables for fields/columns,
   `# Examples` fenced code for usage, `# Citations` (numbered) for sources.
   Define what each field/term *is* and its role — never a bare `name: type`.
   Quote specific values, enum meanings, qualifiers, and formulas verbatim.
3. **Cross-link both ways.** Where this concept relates to another, link it with a
   bundle-relative link — `[customers](/tables/customers.md)` — and add the
   matching link in the other concept too, so the relationship is discoverable
   from either end.

Ground every statement (see the rules below). A thin source yields a short
concept — that is correct, not a failure. Write independent concepts in any
order; nothing depends on a later aggregation step. **Read the worked example
below before your first concept.**

### 5. Generate indexes (deterministic)

After all concepts exist:

```bash
python3 scripts/generate_indexes.py <bundle>
```

Writes an `index.md` in every directory, grouping concepts by `type` and listing
subdirectories. Run it last so it reflects the final tree. Don't hand-write
indexes.

### 6. Validate (deterministic)

```bash
python3 scripts/validate_bundle.py <bundle>
```

Fix every hard `[ERROR]` (missing/unparseable frontmatter, missing `type`, stray
frontmatter in `index.md`). `[warn]` items (missing recommended fields, broken
links, missing index) are spec-tolerated — fix when it improves the bundle.
Broken-link warnings are worth scanning: a typo'd cross-link shows up here.
Use `--strict` only when the user wants a bundle with zero soft issues.

### 7. Report

Report: bundle path, concept count, and validation status (conformant or the
remaining errors). Done.

## After generation

The bundle is plain markdown in a directory — already git-diffable and
Obsidian/Hugo-ready; "export" is just using it. To change it, the user asks in
plain language and you edit the relevant `.md` directly (no special mode):

- "Make the orders overview more concise" → edit that file's body.
- "Add an Examples section to customers" → add the section.
- "Add cascade-delete semantics to the FK" → edit the schema/relationship note,
  keeping it grounded in what the user told you.

Re-run `generate_indexes.py` only if you added/removed concepts or changed a
`title`/`type`/`description`. Re-run `validate_bundle.py` to confirm it's still
conformant. "Score it" = run the validator; it reports the only quality signals
that are actually verifiable (conformance, concept count, broken links). Don't
emit invented quality numbers.

## Grounding rules (these make or break the output)

The whole value of an OKF bundle is that it is *trustworthy* — usable without
re-reading the sources. So:

- **Never invent.** Every statement traces to its source. Padding with plausible
  generalities is the worst outcome.
- **Capture meaning, not just names.** Define what each column/field/term *is*
  and its role. Quote specific values, qualifiers, enum meanings, formulas.
- **Prefer structure.** Tables, lists, fenced code, conventional headings beat
  prose for retrieval.
- **Keep provenance.** Record source URLs/filenames as `resource` and/or
  numbered `# Citations`. Keep real links verbatim; drop dead local-path links.

## Worked example — one source → one concept

**Input** (extracted from `customers.json`):

~~~json
{ "table": "customers",
  "uri": "https://example.internal/catalog/customers",
  "columns": [
    {"name": "customer_id", "type": "STRING", "desc": "Primary key."},
    {"name": "email",       "type": "STRING", "desc": "Null for guest checkouts."},
    {"name": "tier",        "type": "STRING", "desc": "One of free/plus/pro."} ] }
~~~

**Output** — `tables/customers.md`:

~~~markdown
---
type: Table
title: Customers
description: One row per customer account.
resource: https://example.internal/catalog/customers
---

# Schema

| Column        | Type   | Description |
|---------------|--------|-------------|
| `customer_id` | STRING | Primary key. Referenced by [orders](/tables/orders.md). |
| `email`       | STRING | Customer email; **null for guest checkouts**. |
| `tier`        | STRING | Account tier — one of `free`, `plus`, `pro`. |

# Citations

[1] [customers schema](https://example.internal/catalog/customers)
~~~

What the output deliberately **omits**: owner, SLA, row count, lineage — the
source states none, so the concept states none. The `email` qualifier and the
exact `tier` enum are kept verbatim. The FK link is written here and mirrored in
`orders.md`. That is the standard to match.

## Files in this skill

- [references/okf-spec.md](references/okf-spec.md) — the OKF v0.1 specification
  (canonical). Read before planning.
- [references/format-extraction.md](references/format-extraction.md) — how to
  extract clean content from each input format. Read in step 2.
- [scripts/generate_indexes.py](scripts/generate_indexes.py) — generate all
  `index.md` files (step 5).
- [scripts/validate_bundle.py](scripts/validate_bundle.py) — check OKF
  conformance (step 6).
