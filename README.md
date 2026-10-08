# The Aedelgard Mind Format

**Version 1.0 · 2026-10-08 · Status: published specification**

Aedelgard's promise is that your mind is yours and goes wherever you go: memory, identity and
values belong to you, and the model that thinks with them is a rented part you can swap.
A promise like that is only real if anyone can check it. This document defines, in the open,
what a mind *is* on disk, how it travels, how it is sealed, and how the extensions it carries
are signed and checked. The code that implements it may be private; **the format is not.**

Anyone may write a reader, a writer, a converter or a competing product against this spec.
That is the point.

| Part | What it defines | Status |
|---|---|---|
| [1. The mind archive](#1-the-mind-archive) | The zip that holds a whole mind | Built, in every body |
| [2. The sealed envelope](#2-the-sealed-envelope-aedmind2) | How a mind is encrypted for backup and sync | Built (AEDMIND2) |
| [3. The plain export](#3-the-plain-export) | A mind in text, readable without any database library | **Specified; not yet built** |
| [4. Extension packages](#4-extension-packages-aedext) | `.aedext`, the author signature, the review countersignature | Built |
| [5. The catalogue trust chain](#5-the-catalogue-trust-chain) | Root → working key → package, signed index, recall list | Built |

The key words MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.

---

## 1. The mind archive

A mind archive is a zip file. Paths inside it are relative, use `/`, and MUST NOT be absolute,
contain `..`, contain a backslash, or be symlinks.

### 1.1 What travels

Seven **areas** (directories) and two **root files**:

| Path | Holds |
|---|---|
| `config/` | Identity and values (`SOUL.md`, `MEMORY.md`, other prompt files), `mind_id` |
| `memory/` | Daily logs, journal, ledgers |
| `palace/` | The verbatim memory store (see 1.3) |
| `skills/` | Playbooks the mind wrote for itself |
| `commands/` | Commands the mind wrote for itself |
| `tower/` | Local interface state that belongs to the mind |
| `extensions/` | Installed extensions, their shared `data/`, author pins (`authors.json`), the mind's own author key |
| `knowledge_graph.sqlite3` | The temporal knowledge graph |
| `bodies.json` | The machines this mind has lived on |

`config/mind_id` is a 32-hex-digit lineage id, minted once. A restored mind keeps it; a newborn
mints its own.

### 1.2 What never travels

A writer MUST exclude, and a reader MUST ignore if present:

- **Keys and credentials:** `.env`, any keyring file, any private key that unlocks this machine
  (the mind's *extension author* key is the one exception: it travels so every body of the mind
  signs as the same author).
- **Machine-local state:** lock files, process ids, device fingerprints, sync receipts and sync
  state, caches, `local/` folders inside extensions.
- **Recovery and forensic copies:** files and folders whose names begin with `.corrupt-`,
  `.drift-` or `pre_vacuum_backup`; prompt traces; debug output.

The principle: **the mind travels, the locks stay.**

### 1.3 Databases inside the archive

`palace/` holds a ChromaDB store (`chroma.sqlite3` plus its index files) and
`knowledge_graph.sqlite3` is SQLite. A writer MUST copy every SQLite file through SQLite's
online backup API (never a raw copy of a live file), so each one is self-consistent.

**Honest limit:** these are the storage library's own files. Opening them needs a compatible
version of that library. The archive is the fast, complete, same-family format; the
[plain export](#3-the-plain-export) is the format that survives the library.

### 1.4 `MANIFEST.json`

At the archive root:

```json
{
  "mind_id": "…32 hex…",
  "exported_at": "2026-10-08T17:40:00+02:00",
  "body_version": "1.27.50-mind",
  "files": 1234,
  "sqlite_snapshots": 3,
  "largest_file": "palace/chroma.sqlite3",
  "largest_file_bytes": 123456789,
  "contents": ["config", "memory", "palace", "skills", "commands", "tower", "extensions",
               "knowledge_graph.sqlite3", "bodies.json"],
  "envelope": "AEDMIND2",
  "excluded": "keys, credentials, and recovery artefacts"
}
```

Readers MUST ignore unknown keys. A future version adds `"format_version"`; its absence means 1.

### 1.5 Restoring

A reader MUST validate every entry's destination before writing anything, MUST refuse the
whole archive if any entry escapes the target directory or passes through a symlink, and
MUST skip entries outside the areas and root files above.

---

## 2. The sealed envelope (AEDMIND2)

Used when a mind is backed up or synced through a server that must not be able to read it.
The plaintext is a mind archive (section 1).

```
"AEDMIND2" (8 bytes)
salt                      16 bytes, random per seal
chunk_size                uint32 big-endian (writers use 64 MiB)
num_chunks                uint32 big-endian
then, num_chunks times:
  nonce_i                 12 bytes, random per chunk
  ct_len_i                uint32 big-endian
  ciphertext_i            AES-256-GCM output (includes the 16-byte tag)
```

- **Key:** `HKDF-SHA256(ikm = the user's Aedelgard key as UTF-8, salt = salt, info =
  "aedelgard-mind-vault-v2", length = 32)`.
- **Associated data for chunk i:** `"AEDMIND2" || salt || chunk_size || num_chunks || uint32_be(i)`.
  Reordering, dropping, duplicating or splicing chunks from another seal fails authentication.
- The key is derived from the user's own key. A server that stores the envelope holds
  ciphertext only and cannot open it.
- **AEDMIND1** (legacy, read-only): `"AEDMIND1" || salt(16) || nonce(12) || AES-256-GCM(whole
  archive)`, info `"aedelgard-mind-vault-v1"`. Readers SHOULD still open it; writers MUST NOT
  produce it.

---

## 3. The plain export

**Status: specified here; not yet built.** Until it ships, the only export is the archive of
section 1. This section is published first on purpose: the format should be fixed in the
open before the code exists, not reverse-engineered from it afterwards.

A plain export is a zip (or a folder) that any program can read with nothing but a JSON
parser. It carries no embeddings and no database files: vectors are recomputed by whatever
reads it, with whatever embedding model it likes.

```
MANIFEST.json      {"format": "aedelgard-mind-plain", "version": 1, "mind_id", "exported_at",
                    "counts": {"drawers", "facts", "files"}}
drawers.jsonl      one memory per line
facts.jsonl        one knowledge-graph fact per line
files/             config/, memory/, skills/, commands/, extensions/ as plain files
                   (same exclusions as 1.2)
```

**`drawers.jsonl`** — one JSON object per line, UTF-8, verbatim text:

```json
{"id": "…", "collection": "drawers", "text": "the verbatim content",
 "meta": {"wing": "…", "room": "…", "hall": "…", "filed_at": "…", "origin": "…",
          "status": "active", "…": "every other key exactly as stored"}}
```

`status` is one of `active`, `superseded`, `historical`. Superseded and retired memories
MUST be exported: the history is part of the mind.

**`facts.jsonl`** — one triple per line:

```json
{"subject": "…", "predicate": "…", "object": "…",
 "valid_from": "2026-10-08", "valid_to": null, "source": "…"}
```

`valid_to: null` means the fact is still true.

**Round trip:** importing a plain export into an empty mind, then exporting it again, MUST give
the same `drawers.jsonl` and `facts.jsonl` (line order aside). Readers MUST ignore unknown keys.

---

## 4. Extension packages (`.aedext`)

### 4.1 The package

A zip with exactly two kinds of entry:

```
AEDEXT.json        {"format": 1, "name", "version", "kind", "sha256", "files", "signatures": []}
payload/<path>     the extension's files (never data/, local/, or approval state)
```

**`sha256`** is one SHA-256 fed, for each payload file sorted by its POSIX relative path, the
line `relpath \0 hex(sha256(file)) \n`. It binds every file, so a signature over `AEDEXT.json`
binds the whole package.

An importer MUST treat the package as hostile input. It MUST refuse, leaving nothing behind,
any of the following:
- more than 10 MB packed, more than 20 MB unpacked, or more than 500 files;
- any entry outside `payload/`;
- absolute paths, `..`, backslashes, drive letters, symlinks or duplicate entries;
- a shipped `data/` or `local/`;
- a manifest that doesn't match the package;
- files that don't hash to `sha256`.

An imported extension MUST arrive **awaiting approval**, even when it replaces one that was
already approved.

### 4.2 Signatures

Keys are Ed25519, written `ed25519:<base64url without padding>`.
`canonical(x)` = JSON with sorted keys, separators `,` and `:`, ASCII only, and the
`signatures` field removed.

| Signature | Signed bytes |
|---|---|
| Author | `"aedelgard-aedext-author-v1\n" + canonical(AEDEXT.json)` |
| Review (catalogue) | `"aedelgard-aedext-review-v1\n" + canonical(AEDEXT.json) + "\n" + author_public_key` |

Because the review covers the author's key, a review can't be moved onto another author's
package. A signature that is present but doesn't verify is **tampering** and MUST refuse the
import. It is never treated as "unsigned". A review by a key the reader doesn't know means
"not reviewed", not tampering.

**Author pins belong to the mind** (`extensions/authors.json`). A later package with the same
name but a different author key is a different author, never an update.

**What approval means, and what it does not.** Approval pins the exact files: change one
byte and it asks again. That is *integrity*. It is **not isolation**: an approved code
extension runs with the rights of the program that loads it. A review countersignature
means a human at Aedelgard read it. It is not a sandbox.

---

## 5. The catalogue trust chain

- **Root key** (`root-1`): its public key ships inside every body. It signs only delegations
  and, in an emergency, the recall list. Replacing a root needs a new body release.
- **Working key** (`cat-N`): reviews packages and signs the index. It is trusted through a
  **delegation** `{v: 1, key_id, public, issued, purpose: "catalogue-review", issuer, sig}`, which
  the root signs over `"aedelgard-catalogue-delegation-v1\n" + canonical(delegation minus sig)`.
- **Signed documents** look like `{payload, key_id, delegation?, sig}`. Each is signed over its
  document's own domain string followed by `canonical(payload)`.
- **`index.json`**: payload `{format: 1, generated, extensions: [entry]}`. Each entry records,
  among other things, the package's `url` (relative, same base only), `size` and `sha256`.
- **`revoked.json`**: payload `{format: 1, generated, keys: [key_id], packages: [{name, version?,
  reason}]}`. A recalled package is disabled, with the reason shown. Extensions a user made by
  hand are never touched.
- Live at `https://aedelgard.com/catalogue/`. Verification is offline. Fetching reveals only that
  a body exists.

---

## Versioning and change

- Every format named here carries its own version number. A change that breaks readers gets a
  new number; old numbers stay readable for at least one year.
- **Changes are made here first,** in public, before any body ships them.
- **Open code:** section 4 (packages and signatures) is implemented in the open engine,
  [avasol/galadriel-public](https://github.com/avasol/galadriel-public), tag
  `reference-2026-10-08` (`harness/ext_signing.py`, `harness/extensions.py`). Sections 1, 2 and 5
  are implemented in the Aedelgard body, whose code is not public. This document, not any
  code, is the source of truth. A body that disagrees with it has a bug.

## License

This specification is published under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
Implement it freely.
