# anything-to-okf — Convert Any File Format to Open Knowledge Format

**Turn your documents, data, and code into a linked knowledge bundle — pure
markdown, no vendor lock-in.**

A Claude Code skill that converts **any file format** (CSV, PDF, Word,
PowerPoint, Excel, JSON, HTML, Markdown, source code, images) into an **Open
Knowledge Format (OKF)** bundle: a directory of markdown files with YAML
frontmatter, cross-links, and `index.md` listings — readable by humans,
parseable by agents, diffable in git.

Good for:
- 📊 Building a knowledge base from a folder of mixed sources
- 🔗 Linked data catalogs (tables, schemas, docs)
- 🤖 Feeding agents structured, portable knowledge
- 📝 Publishing to Obsidian, MkDocs, Hugo, or git

---

## How it works

The skill is **instructions to Claude**, not a pipeline of subprocesses. Claude
does the conversion directly:

1. **Discover & classify** the input files.
2. **Extract** each source to faithful text/markdown (office formats via the
   bundled `pdf`/`docx`/`pptx`/`xlsx` skills).
3. **Plan** the concept set, directory tree, and `type` for each concept.
4. **Write** each concept's `.md` directly — grounded in its source, with
   bundle-relative markdown cross-links between related concepts (added on both
   sides of a relationship).
5. **Generate indexes** — `scripts/generate_indexes.py` builds every `index.md`.
6. **Validate** — `scripts/validate_bundle.py` checks OKF conformance.

Only steps 5–6 are scripts, because they must be deterministic. The judgment —
what the concepts are, how they link, what each one says — is Claude's, done
inline. No CLI dispatch, no hidden "merge pass", no invented quality scores.

---

## Installation

**Prerequisite:** [Claude Code](https://claude.ai/code), and Python with PyYAML
(`pip install pyyaml`) for the two scripts.

**Plugin marketplace:**

```
/plugin marketplace add CAgcoder/anything-to-okf
```

**Manual:** clone this repo and copy `SKILL.md`, `scripts/`, and `references/`
into `~/.claude/skills/anything-to-okf/`. Restart Claude Code to activate.

---

## Quick start

In any project directory, just ask:

```
Convert my-data/orders.csv my-data/customers.json docs/schema.md to OKF
```

or `build an OKF bundle from this folder`. Claude extracts the sources, plans the
concepts, writes the bundle, generates indexes, and validates it:

```
✅ okf-bundle/ — 12 concepts, OKF-conformant (0 errors, 2 warnings)
```

To change anything afterward, just say so — the bundle is plain markdown, so
Claude edits the relevant file directly:

- "Make the orders overview more concise"
- "Add an Examples section to customers"
- "Score it" → re-runs the validator (conformance, concept count, broken links)

There is no separate refine/score/enhance/export tooling — the bundle is already
git- and Obsidian-ready, and edits are ordinary file edits.

---

## What is OKF?

**Open Knowledge Format** is a vendor-neutral markdown+YAML convention for
knowledge bundles:

- **Human-readable** — just markdown files in a directory.
- **Agent-readable** — YAML frontmatter + plain markdown cross-links.
- **Portable** — no lock-in; works with git, Obsidian, MkDocs, Hugo.
- **Minimal spec** — only a non-empty `type` field is required; everything else
  is optional and consumers must tolerate its absence.

See [references/okf-spec.md](references/okf-spec.md) (short read).

---

## Repository layout

```
SKILL.md                       # The skill: instructions to Claude
scripts/
├── generate_indexes.py        # Build every index.md (deterministic)
└── validate_bundle.py         # Check OKF conformance (deterministic)
references/
├── okf-spec.md                # OKF v0.1 spec — read before planning
└── format-extraction.md       # How to extract from each file type
evals/                         # Fixtures + test cases
```

---

## License

MIT — see [LICENSE](LICENSE).
