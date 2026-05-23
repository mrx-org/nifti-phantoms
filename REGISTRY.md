# NIfTI Phantom Registry

[`registry.json`](registry.json) is a public, PR-editable index of NIfTI
phantoms. It only **references** data — the phantoms themselves are hosted on
[Zenodo](https://zenodo.org/). Anyone can publish a phantom and add
it via a pull request. Entries are validated against
[`nifti-registry.schema.json`](nifti-registry.schema.json) (JSON Schema
draft 2020-12). The format carries no version tag and entries allow extra
properties, so it can be migrated in place if it ever needs to change.

## Hosting model

A phantom consists of large, static **NIfTI** files and a small **JSON** file
that is tweaked more often (new field strengths, revised T1/T2, …). Both are kept
in **one Zenodo record**. Each published version of that record gets an immutable
**version DOI**: two people citing the same version DOI are guaranteed
byte-identical files. The version DOI **is** the integrity guarantee, so the
registry stores no checksums.

Every entry pins its `doi` to one such immutable version. There is
deliberately **no concept DOI** — following the latest version is the registry's
job, not the record's. The entry is **mutable**: when a phantom is revised (a new
Zenodo version is published) the entry is updated in place to the new version DOI
via a PR. So the registry always points at current data, while each `doi` it
points at is itself frozen.

A Zenodo version DOI is `10.5281/zenodo.<record_id>` — it embeds both the host
and the record id, so the `doi` alone is enough to resolve and download the
files. The registry therefore stores no `provider` or `url`: both are implied by
the DOI. (Zenodo is the only host for now; another could be added later in place
if requested.)

You iterate freely (locally / in a fork), then *freeze* by publishing a Zenodo
version. Zenodo's **"New version"** flow carries unchanged files forward without
re-uploading them, so revising the JSON does not re-upload the NIfTI.

A record may bundle a **set** of related phantoms (e.g. one subject across field
strengths, or a whole cohort). One registry entry — a **collection** — maps to
exactly one record and lists every phantom JSON in it. A record's phantoms are
therefore never split across entries, and no two entries share a `doi`.

### Reproducibility

Because entries are mutable, the immutable unit is the **registry itself**, which
is versioned by git. To pin a phantom in your code, reference it as
`<collection>/<file>` together with the **nifti-phantoms commit hash** of the
registry you resolved against — e.g. `a1b2c3d mrx-brain-cohort/subj42-3T.json`.
That commit fixes the collection's `doi`, and therefore the exact files, forever.
Resolve against the registry's `main` HEAD for the newest data; pin to a commit
for a reproducible resolution.

## Entry format

`registry.json` is a top-level array of **collections**. Each entry:

| Field | Req. | Description |
|-------|------|-------------|
| `collection` | yes | Unique collection name; namespaces its files in references (`mrx-brain-cohort`). |
| `description` | yes | One or two sentences describing the collection. |
| `authors` | yes | List of `{ name, orcid?, email?, affiliation? }`. |
| `license` | yes | SPDX id, e.g. `CC-BY-4.0`, `CC0-1.0`. |
| `doi` | yes | Immutable Zenodo **version DOI** (`10.5281/zenodo.<id>`) the collection points to. Updated in place (new PR) when the data is revised. |
| `phantoms` | yes | The phantom JSON filenames in the record (≥ 1). |
| `keywords` | no | Discovery tags (`brain`, `synthetic`, `3d`, …). |

`phantoms[]` is a flat list of JSON filenames inside the record (e.g.
`subj42-3T.json`). Each is referenced as `<collection>/<filename>` and pulls in
the NIfTI files it needs from the same record. Names should be self-describing
(field strength, options) since there is no per-phantom description.

Resolving a phantom: read the Zenodo record id from the collection's `doi` (the
digits in `…/zenodo.<id>`), then fetch files from the record over the Zenodo API —
download the chosen `phantoms[]` JSON and the NIfTI files it references (all live
in the same record). To follow the latest data, resolve from the registry's
`main` HEAD; to reproduce an exact resolution, resolve at a pinned commit.

Deliberately minimal: tissue lists, resolution and channel counts are discovered
by opening the phantom JSON, not duplicated here.

## Contributing a collection

1. Assemble the phantom set (NIfTI + JSON) following the layout in
   [SPEC.md](SPEC.md).
2. Upload **all files** to a single Zenodo record and publish it. To revise the
   JSON later, use Zenodo **"New version"** and keep the existing NIfTI files.
3. Open a PR adding one collection entry to [`registry.json`](registry.json):
   list every phantom JSON in the record under `phantoms` and set `doi` to the
   published version DOI. To revise later, publish a new Zenodo version and open
   a PR that updates the collection's `doi`.

A GitHub Action validates `registry.json` against the schema on every PR. Run the
same check locally before opening one:

```sh
pip install jsonschema
python tools/validate_registry.py
```
