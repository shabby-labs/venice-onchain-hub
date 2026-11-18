# venice-onchain-hub

A [Research Commons](https://github.com/research-common/research-commons) hub (1.0 format, tool pinned at
`v1.0.0-rc.1`) for **Venice on-chain data on Base**: captured chain data, the workflows that derive results from
it, and those results, each one signed.

> **Pilot hub.** This hub is maintained by its publishers (shabby-labs). The Research Commons project doesn't endorse it,
> and its contents are not reference data. As with every hub, what you trust is your own reader policy.

## What is here

| | |
|---|---|
| Publisher identity | fixed-root identity `57f2d9f6db5cd5536a8805218f294f486481ed228f861f490d3a3e3f658b4441` (root on a hardware device); statements are signed by a worker key acting for it under a grant |
| Collection | **Venice DIEM staking (Base)**, resource `61851d1a3b7e5f9379d64928284b687da730cbbe32656e33939c1c62c092fe99`, curated |
| Dataset | `diem-stake-events`: every DIEM `Staked`/`Unstaked` event from the staking contract's deployment through Base block 58,900,000 (30,091 rows, NDJSON), `observed`, class `exact-bytes`, licence CC0-1.0 |
| Analysis | *DIEM staking activity by window*: a pinned workflow (`alpine:3.20` by digest, no network) that counts stakes/unstakes, sums wei and counts distinct addresses per 1,296,000-block window; output CC-BY-4.0, code Apache-2.0 |
| Selection | curated by the publisher; the collection's editions say what is a member |

```bash
git clone https://github.com/shabby-labs/venice-onchain-hub && commons init mystore && cd mystore
commons pull ../venice-onchain-hub --from-checkout
commons show 61851d1a3b7e5f9379d64928284b687da730cbbe32656e33939c1c62c092fe99
commons verify --rerun --exec-policy <your policy> sha256:7181144263925f455ba2133f1f47046f32b66e61cf806102108f9d6a8f92cbc1
```

**Reproduce it.** The analysis reads `sole author · not re-derived` until a second, independent key re-runs the
workflow on the same input and publishes the same bytes (then `T0`). Independent re-derivations and
independent captures of the same block range are welcome as pull requests.

### Known display issue (1.0.0-rc.1): the dataset's licence reads `unknown`

`diem-stake-events` is an `observed` capture of public chain data. Its signed statement declares
`licence: CC0-1.0`, `replication: bytes` and `availability: permitted / obtainable`, and those fields are
in the signed bytes. The 1.0.0-rc.1 `commons show` / `list` / `explain` builds single-party defaults from
`authored` statements only, so for an `observed` blob it prints `licence: unknown`. This is a display gap in
the tool, not a missing declaration, and it needs no republishing: a reader with the fix will show CC0-1.0 for
the same statement. Until then, read the statement body (`commons show <statement id>`). The derived CSV, the
workflow and the step code are `authored` and display their licences normally.

---

*The rest of this README is the hub template's general documentation.*

## What is in a hub

```text
rc-store                         layout marker: {"layout":"rc-store/1"}
blobs/sha256/<64 hex>            content: the bytes whose sha256 is the file name
statements/<signer>/<id>         one signed statement per file; the file name is the statement id,
                                 the directory is the signer (":" written as "-")
hub-policy.json                  this hub's admission policy (control plane, see below)
```

That is the whole data surface. There is no shared file that anyone edits:

- A **blob** has no owner and no metadata. Two people who publish the same bytes share one blob.
- A **statement** is one key's signed word about something: that it authored some bytes, observed them,
  reproduced them, holds a copy, retracts them; an edition of a collection; a time anchor. Each key's
  statements form its own log, chained by the predecessor each one names inside its signature. Nobody can
  add to, edit or remove another key's log.
- **Anchors** are statements too (`anchors`), with the timestamp proof stored as a blob. They need no
  directory of their own.
- **Collections, tasks and growing series** are resources: a genesis statement and signed editions, in their
  controllers' logs. Their membership rules travel inside the editions.
- **Indexes, caches, grades and views** are not stored. Every reader computes them from the blobs and
  statements it holds, under its own policy. Deleting them changes no answer.

A hub may also hold **no blobs at all**: a `holds` statement can point at bytes stored elsewhere (an object
store, a content-addressed network, a plain URL), and a reader verifies the full hash whatever the source.
Set `"blobs": "external-only"` in `hub-policy.json` for that configuration.

## Start a hub

1. Create a repository from this template.
2. Edit `hub-policy.json` if the defaults do not suit you (`docs/hub-policy.md` explains each field). Commit
   it on the default branch.
3. Pin the tool in `.github/workflows/hub-check.yml` (see *Upgrading the tool pin*).
4. Protect the default branch: require the `hub-check` job and a review before merging. A pull request that
   changes only control-plane files (the policy, this README, the workflow) passes `hub-check` with a
   notice, because CI has nothing to admit; the review is what guards those files. A `CODEOWNERS` entry for
   `hub-policy.json` and `.github/` makes that review explicit.
5. Say in this README what the hub is for and how it selects what it admits: curated by its maintainers,
   by a published rule anyone can re-apply, or a completeness claim others can check. This is prose for
   people. The tool shows only a selection someone signed (a collection declares one in its editions), so a
   hub as a whole displays as `undeclared`, which readers treat as curated.

## Read it

```bash
git clone <this hub> && cd <this hub>
commons verify <id>          # re-check an artifact against the statements you hold
commons explain <id>         # the statements, policy and checks behind a status
```

Every status a reader sees is scoped: "corroborated by 2 *within the statements you hold*". A hub is one
source; a reader who follows several sees what each holds, including contradictions a single hub left out.

## Contribute

1. Generate a signing key (the tool does this on first run).
2. In a fork or branch of this hub, publish with the tool. It writes your statements under
   `statements/<your key>/` and the bytes under `blobs/sha256/`. Commit only those.
3. Open a pull request. CI runs `commons hub check --base <target>` and reports a verdict for every object.
4. A maintainer merges, or ingests your branch on their own machine with `commons pull`, which also applies
   their own trust.

You never edit an existing file in a hub. To correct something, publish a new statement: a later
`authored` that supersedes your earlier bytes, a `retracts`, a new edition.

## What CI checks

Each object in a pull request goes through the tool's one validation core, the same function `pull` and
`push` use. In outline (the full table is `docs/CLI-CONTRACT.md`):

- **Only data paths change.** A pull request may add files under `blobs/` and `statements/` and nothing
  else. It may not modify, rename or delete anything already held. Changes to `hub-policy.json`, this
  README, the workflow or any other file are reported as control-plane changes for a maintainer to review,
  and are never applied as part of a data change.
- **Bytes match their names.** Every blob hashes to its file name; every statement's id is its file name.
- **Signatures and form.** Every statement is canonical, signed by the key its directory names, and well
  formed.
- **This hub's policy, from the base revision.** The check reads `hub-policy.json` as it is on the branch
  being merged into, never from the pull request. A pull request that loosens the policy and adds something
  only the looser policy admits fails.
- **Collection rules.** Where a statement says which collection edition it was checked against before
  publishing, CI re-applies that edition and the current one, and shows both.

What CI admits with a label rather than refusing: statements from a tool version newer than the checker
(`unsupported`), statements whose predecessor the hub does not hold yet (`detached`), a key whose log has
forked (`FORK`), bytes held elsewhere (`unavailable`), and every disagreement between keys (`contested`).
None of those is a reason to refuse valid signed data.

Exit codes: `0` everything would be admitted · `1` at least one object refused, each named with its reason
and scope · `2` the check could not start (bad option, unreadable policy) · `3` admission of an object
could not be decided. Every non-zero code fails the job.

## What CI deliberately does not decide

- **Who is trusted.** CI does not check whether a signer is a maintainer, is known, or may publish here. Trust
  is each reader's own policy and stays on their machine; maintainers apply theirs when they `commons pull`.
  A valid statement from an unknown key passes CI; whether anyone believes it is a separate question.
- **Whether anything is true.** Signatures prove who said something, not that it is right. Corroboration,
  grades and contradictions are computed by each reader.
- **Time.** CI checks that an anchor statement is well formed, not that its proof is genuine. Each reader
  verifies timestamp proofs against block evidence it trusts.
- **Execution.** CI never runs anything from the hub. A reader decides for itself what it executes, and
  runs foreign code sandboxed.
- **Completeness.** A pull request is a package of objects. It claims nothing about what else exists.

## Removing content

A hub never deletes statements, including retractions. A retraction is displayed wherever the artifact is
shown; it does not by itself stop anyone serving the bytes. If this hub must stop serving some bytes, add
their digest to `not_served` in `hub-policy.json` (one merge), then remove the blob (a second merge, which
CI admits because the base policy now lists it). Statements about the content remain, and readers see it as
not held here. Deleting the bytes here does not erase them anywhere else.

## Coming from an alpha hub

The next format is a clean break. There is no importer and no dual-format reading, and the tool refuses an
alpha hub tree (`registry/`, `store/`, `.commons-hub`) as malformed rather than guess at it.

- **Content carries over unchanged.** The same bytes have the same sha256, so you do not regenerate data;
  you republish it.
- **Statements are new.** Authorship, licences and time evidence are re-signed and re-anchored under the new
  format. Priority established under the alpha does not carry over.
- **Start a new repository** from this template and republish into it. Keep the alpha hub as it is, for
  reference; do not convert it in place.

## Upgrading the tool pin

`hub-check.yml` builds the tool from a **full commit SHA**, with the release tag as a trailing comment, so a
change upstream cannot alter what this hub accepts until you choose to move the pin. To upgrade:

1. Pick a release and resolve its tag to a commit:
   `git ls-remote https://github.com/research-common/research-commons.git 'refs/tags/<tag>^{}'`.
2. In a pull request, set `TOOL_REF` to that SHA and update the tag comment.
3. Before merging, run `commons hub check` with the old and the new tool on this hub's default branch. A
   check that newly refuses held data needs a decision before you merge, not after. Read the tool's
   `CHANGELOG.md` for the versions in between.

The CI log prints the tool commit and `commons --version` on every run.

## Maintainers

How a hub names its maintainers is not settled for the next format yet. Until it is, list them here in
prose. A list in this file grants nothing to anyone: CI does not read it.
