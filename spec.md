# Folder Object Format 1.0 — Structure Layer

**Status:** Draft — 2026-09-04, the DASL encoding branch: the hash encoding is [DASL](https://dasl.ing/) (DRISL + MASL) instead of UnixFS; everything else is the 2026-08-24 draft. Every root changes (`bafyrei…` where it was `bafybei…`), `ipfs add` no longer reproduces it, every block remains an IPLD block. Not frozen; a prototype of the encoding exists (`@estoc/folder-object/dasl`, estoc branch `dasl`). Subject to incompatible change. Appendix A lists what moved and what a switch touches.

## Abstract

This document defines a content format whose only substrate is a plain file tree. An **object** is a tree of the shape `{index.json, files/}`: a pure fact with no container conventions and no exclusion rules. An object's version identity is the content hash of its canonical tree; its entity identity is a UUID carried in the index. Signing happens outside the tree: a **signed object** pairs the object with a detached card that means one thing — this DID stands behind this object. Signatures cover the logical tree, never container bytes.

## Scope

Version 1 of this specification covers **internal files only**: an object's own index and its own bytes.

It does not cover cross-object references — the `objects` dependency table (immutable and mutable references, inlined dependencies, deferred location hints). These are introduced in a later version of the format family; the names they will occupy are reserved by this document (§3.2, §5).

Also out of scope: the vocabulary contracts of individual types (e.g. the field requirements of a post — defined per type format, see §3.1), the choice of body markup language, and the exact shape of the share message that carries objects over DIDComm.

## 1. Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals.

**Fact.** A fact is a mapping from relative paths to byte strings — nothing more. It carries no timestamps, no permissions, no empty directories, no symbolic links. Directories exist only as prefixes of file paths: a container that holds an empty directory reproduces the same mapping as one that does not, so an empty directory is never part of an object and never enters its hash (the hash encoding below cannot even express one: a manifest lists files, nothing else); and no path is a directory of another (`files` beside `files/x` is not a mapping a tree can hold).

**Path.** A path is a sequence of non-empty segments joined by `/`. A segment is a Unicode string, carried as UTF-8; it contains neither `/` nor U+0000, and is neither `.` nor `..`. Paths are compared as their UTF-8 bytes — never percent-encoded, no normalization, no case folding: two names that differ only by normalization or by case are two paths, and a container that cannot keep them apart cannot reproduce the fact (§8).

**Container.** Any encoding from which the path → bytes mapping can be reproduced (a directory on disk, a zip file, a set of DIDComm attachments) is a legal container. **No container has normative status**; all semantics attach to the mapping.

**Hash encoding.** Hashes are [DASL](https://dasl.ing/). A **DASL CID** ([CID](https://dasl.ing/cid.html)) is a CIDv1 over sha-256 with the codec `raw` (0x55) or `drisl` (0x71): 36 bytes, `01 <codec> 12 20` and the digest — only that hash (BDASL's BLAKE3, 0x1e, is malformed here) and only those codecs. Its string form, everywhere in this format — a card's `root`, a share's root, an attachment's name, a blob's name — is the one spelling `b` followed by 58 characters of the lowercase base32 alphabet, unpadded, 59 characters in all; any other spelling of the same bytes is malformed, not an alias. A **DRISL** document ([DRISL](https://dasl.ing/drisl.html)) is deterministic CBOR — the CBOR/c-42 profile: shortest forms, definite lengths, text keys in bytewise order of their encoding, CIDs as tag 42, nothing else — so that one value has one byte string and therefore one CID. A file's bytes are named by the raw CID of the whole of them, never chunked. A tree is named by the drisl CID of its **manifest** (§2.1): one DRISL document, in the shape of a [MASL](https://dasl.ing/masl.html) bundle, mapping every path of the fact to the CID and size of its bytes. There are no directory nodes: a fact is a flat mapping, and so is its encoding. One hash encoding serves the whole stack, and any DASL implementation can recompute or verify an object's version identity from the mapping alone — a few hundred lines over sha-256; every block is an IPLD block an IPFS node can store, fetch and check by CID, though `ipfs add -r` does not produce this root. The codec tells a manifest from a file: a drisl CID roots a tree, a raw CID names bytes.

## 2. Object Format

An object is a tree with exactly two things at its root:

```
<object>/
  index.json        # the object's index (§3)
  files/            # the object's own bytes (§4)
```

**Canonical tree.** The canonical tree of an object is `index.json` plus the entire `files/` subtree, less hidden entries (§4). It is taken by enumeration; the hidden-entry rule is the one name-based rule, and it is this format's (§4). There are no out-of-tree entries and no other exclusion rules inside an object: **an object is pure fact**. The card stands beside the object (§5), not inside it.

**Identity.**

- **Version identity** = the **root**: the manifest CID of the canonical tree (§2.1) — value semantics.
- **Entity identity** = the `id` field of the index (UUIDv7; stable across versions).

Entries outside the canonical tree that happen to share a container with an object (drafts in a vault, editor litter) are legal but are not part of the object.

**Identity is packaging-neutral** (core invariant). Whether a card is present, and how the object travels, are decided entirely outside the object's folder; the root CID is structurally unaffected. Packaging is a transport decision, never a semantic one.

### 2.1 The Manifest (Hash Encoding)

The **manifest** of a mapping is the DRISL document

```
{ "resources": {
    "/index.json":              { "src": <raw CID>, "size": 272 },
    "/files/body.md":           { "src": <raw CID>, "size": 88 },
    "/files/images/sunset.png": { "src": <raw CID>, "size": 70 } } }
```

with exactly one entry per path of the mapping, in DRISL's key order; the **root** is the drisl CID of the manifest's bytes.

The manifest is the hash encoding of the canonical tree, not a file of it: every hasher computes it from the mapping, no writer authors it, and it never appears in the object folder — a file so named there is ordinary bytes under `files/`, or litter beside `index.json`. It travels and is stored as a block named by the root (§7), and a reader learns nothing from it that the leaves do not prove.

- **Keys** are `/` followed by a path (§1). The manifest of an object names exactly the canonical tree — `/index.json` and paths under `/files/`, none hidden (§4) — and nothing else: a root that reaches litter, a card, or a hidden file was not computed over an object, no canonical tree hashes to it, and a reader MUST refuse it rather than filter it (§8). A key that is not `/` plus a path a tree can hold — an empty segment, `.` or `..`, U+0000, a key that is a directory of another (`/files` beside `/files/x`: MASL allows it, a tree cannot hold it) — was not computed over a fact at all (§8).
- **`src`** is the raw CID of the file's bytes, as a CBOR tag 42 link (the byte string `0x00` ‖ the 36 CID bytes). It MUST be a `raw` CID: a manifest never links a manifest — objects refer to objects through the `objects` table (§11), not through the tree. **`size`** is the byte length of the file: a CBOR unsigned integer no greater than 2^53 − 1, and every entry naming one `src` states one `size`. For a present leaf, `size` is checked against the bytes, and a disagreement means the manifest was not computed over this mapping (§8). For an absent leaf it is the signer's claim: a reader MAY display it and MUST NOT allocate or reserve on it.
- **Nothing else.** An entry has exactly `src` and `size`; the document has exactly `resources`. The manifest is derived, never authored, so it is closed like the card (§6), not open like `index.json` (§3.2): two encoders that disagreed about an extra member would give one fact two roots. In particular it carries no `content-type` (the index declares the content's media type, other files are typed by extension, §4), no `name`, no `/` entry, no `version`/`roots`, no `prev`; MASL's invitation to add namespaced metadata at the top level is refused here, explicitly. The manifest is **closed and unversioned**: exactly these members, in this form, for every version of every type format in the family. Anything an object needs to say lives in `index.json`; a different hash encoding would be a different structure specification, not a version of this one, and its roots would carry a different codec or be rejected.
- **Canonical bytes.** The manifest has exactly one encoding:

  ```
  manifest = A1 69 "resources" head(5, n) entry×n          entries strictly ascending by the bytes of their encoded key
  entry    = head(3, len) "/" path                          the key: a text string, valid UTF-8
             A2 63 "src" D8 2A 58 25 00 cid                 tag 42 over 0x00 ‖ the 36 CID bytes
                64 "size" head(0, size)                     an unsigned integer
  cid      = 01 55 12 20 digest(32)
  root     = 01 71 12 20 sha-256(manifest)
  head(t, n) = t<<5 | n (n < 24) | t<<5|24, n (< 2^8) | t<<5|25, n (< 2^16) | t<<5|26, n (< 2^32) | t<<5|27, n (< 2^64)
  ```

  Every head in its shortest form; definite lengths only; keys ordered by the bytes of their encoding (length first, then bytewise — never a string order); no tag but 42, no float, no simple value, no byte after the top-level item; the whole no larger than 1 MiB. This is DRISL as DASL specifies it (the CBOR/c-42 profile), written out so that two encoders agree byte for byte without reading a draft; `@ipld/dag-cbor` and `@atcute/cbor` produce these bytes for these values. A reader MUST reject a manifest block whose bytes are any other form — a non-shortest integer or length, an indefinite length, keys out of order or repeated, another tag, a float where an integer belongs, invalid UTF-8, trailing bytes: no conforming hasher produces other bytes for a mapping, so a root over other bytes is not any object's identity. A decoder that cannot tell every non-canonical form apart MUST re-encode what it decoded and require equality.
- **The whole listing is one block.** The manifest is the tree's complete skeleton — every path, its size, its CID. A reader holding the manifest and `index.json` knows what the object is and what it contains before any other byte arrives, and resolving a path is two fetches, whatever the depth. A manifest larger than 1 MiB is malformed: it must fit the skeleton's inline budget (§7), and the bound is what a reader decodes before it trusts anything — about ten thousand entries.
- **Files are whole.** A file is exactly one raw block over its entire bytes, whatever its size: version 1 defines no chunked or multi-block encoding of a file (DASL expects the same of its CIDs, informatively; here it is the rule), so there is no chunk layout to agree on. The ceiling is the reader's: a leaf is verified by hashing all of it, so a reader MAY decline a leaf whose `size` exceeds what it can hold — the object is then unverifiable by that reader, not malformed — and MUST bound every fetch by its own limit, never by `size`. A chunked file (a DRISL chunk list under `src`, or DASL's BDASL with streaming verification) would be a breaking change to this closed manifest, deliberately deferred (§12).
- **MASL.** Every object manifest is a valid MASL bundle-mode document, and the root is built the way DASL's tiles build a tile's CID — the drisl CID of the MASL document's bytes. The conformance is one-directional: this format adds what MASL lacks (the closed member set, paths a tree can hold, `/index.json` required, `src` raw only), so not every MASL bundle is an object manifest; and its keys are the mapping's paths byte for byte, not URL pathnames — MASL's URL-resolution rules do not apply to them. A MASL processor ignores `size` and, finding no `/` entry, renders nothing on its own: an object is not a web page, though a projection of one (§10) may be published as one.
- The empty mapping has a manifest (`{"resources": {}}`, root `bafyreiarjrxb4yyyuxufubktb6de267lxmqvipdyk5dffbqjnvidwncvau`) but is not an object (§8: no `index.json`).

## 3. index.json

```json
{
  "format": "https://estoc.dev/post/1.0",
  "id": "01a03110-7c1e-7b3a-9f42-3d5e8a1b2c04",
  "content": { "mediaType": "text/markdown", "path": "files/body.md" }
}
```

`index.json` MUST be a JSON object.

### 3.1 Structural members (family contract)

The members defined here form the **structural contract shared by every type format** in the family.

- `format` — REQUIRED. A **type format URI**, including a version, identifying what kind of tree this is and which vocabulary contract it follows (e.g. `https://estoc.dev/post/1.0`). Its semantics align with the `format` field of DIDComm attachments: it names a concrete format, not the generic fact of being an object tree. The structure layer itself (this document) never appears as a wire value. Renderers dispatch on `format`.
- `id` — REQUIRED. A UUIDv7 string; the object's entity identity.
- `content` — OPTIONAL. The object's principal bytes, in one of two forms:
  - **in-tree**: `{mediaType, path}` — `path` MUST point into `files/`;
  - **inline**: `{mediaType, text}` — in this form `files/` MAY be absent (an empty `files/` is not a thing: the fact has no empty directories), so **a lone `index.json` is a complete object**.

  Whether `content` is required is decided by each type's vocabulary contract (a post will require it).

A type's own vocabulary members (`title`, `published`, `updated`, `summary`, `tags`, …) are orthogonal to the structural members and live in the same `index.json`, as defined by that type's format document.

### 3.2 Generic processing and forward compatibility

- **Generic machinery does not read the value of `format`.** Hashing, closure computation, signing, and transport are all shape-driven: the canonical tree is taken by enumeration and the closure check reads only `content.path`. An object with an unknown `format` MUST still be storable, transportable, and verifiable; rendering falls back to a generic card.
- Readers MUST ignore unknown top-level members of `index.json`. (Vocabulary members and future structural versions layer on here.)
- `objects` is a **reserved name**. Version-1 writers MUST NOT emit it; version-1 readers MUST NOT interpret it. A future object carrying a dependency table therefore remains a well-formed object to a version-1 reader — its dependencies are simply invisible, and body references to them degrade to broken links.

## 4. `files/` — the Object's Own Bytes

- The entire `files/` subtree is the object's own bytes: **all of it enters the root hash, all of it is covered by the card, and none of it is declared per file.** Closure computation depends on no body parser: the canonical tree is itself the complete closure.
- Bytes in `files/` are **values**: they have no `id` and no independent life; their identity is absorbed by the object's root hash.
- Version 1 defines only the trivial encoding: paths inside `files/` are real paths, and directory structure carries no semantics (layout is the object's private business).
- **Hidden entries are not part of the tree.** A path is hidden iff any of its segments begins with `.`. The rule is a function of the name alone (the mapping has no platform attributes), so `files/.DS_Store` and `files/.git/…` never enter the manifest, and the same folder hashes to the same root wherever it is read. A `content.path` with a hidden segment can never have bytes and is malformed (§8).
- **A fact has no symbolic links.** A container that holds one cannot reproduce a path → bytes mapping for it; a reader MUST refuse such a container (never follow the link, never silently drop it). The manifest has no way to say "link", and this format gives it none: a folder with a symlink is not an object at all, rather than an object with a different hash.
- **Body references** are resolved by the body's own media type (Markdown, HTML, … each already define how a relative reference resolves against the document); this format only fixes the two things a media type cannot know. **Base:** a body given as `path` is located at that path (`files/body.md` resolves `images/fig1.jpg` to `files/images/fig1.jpg`); a body given as `text` is located at the object root. **Boundary:** a reference that resolves outside the tree, or to a path with no bytes, is a **broken link** — a rendering-layer placeholder, not a malformed object. An absolute `http(s)` URL in the body is an ordinary external link and MUST NOT be loaded automatically (under end-to-end encryption, an external fetch leaks the reader's identity); it opens on click.
- Media types: `content` is declared by the index; other files under `files/` are typed by extension.

## 5. Signed Object

A signed object is the value of `sign(object, card)`:

```
<signed>/
  object/           # the object's canonical tree, verbatim (object/index.json, object/files/…)
  card.jws          # the card
```

- **Recognition.** A tree is a signed object if and only if `object/index.json` exists and is well-formed and `card.jws` is beside it. There is no other container magic: format identity rests on in-tree shape.
- **Litter.** Entries other than `object/` and `card.jws` are not part of the signed object and MUST be ignored by readers — a rendered page, a zip of the same, may sit beside the fact. Writers MUST NOT rely on any such entry being read.
- **Not a package of dependencies.** A signed object carries one object. References to other objects (the `objects` binding table, reserved for a later version) point at objects that are each signed on their own; they are never copied in. `objects/<name>/` and `links.json` remain **reserved entry names** of the layout: version-1 writers MUST NOT produce them; version-1 readers MUST ignore them.
- Unsigning — taking `object/` out — is **identity-neutral** (§2 invariant, made structural): the object is the same object with or without a card.

## 6. The Card

- The card is a compact JWS, `alg: EdDSA`, `typ: estoc/object-card`, `kid` naming the signer's verification method, over the payload `{did, root}`, where `did` is a `did:key` and `root` is the manifest CID of the object's canonical tree (§2.1). A JWS without that `typ` is not a card; a `kid` that is not the payload's `did` makes the card malformed; so does a `root` that is not the canonical string of a drisl DASL CID (§1) — a raw CID names bytes, and this format defines no signature over bare bytes; a `bafybei…` root is a UnixFS-era identity that nothing in this version computes. The payload is the JSON text `{"did":"…","root":"…"}` — **exactly** these two members, each once, in that order, without whitespace: a card with any other payload is malformed, so that no two parsers can disagree about what one signature attests. (This is the opposite of `index.json`, whose unknown members are ignored: content must be able to evolve, testimony must stay closed. Anything new is a new `typ`, not a new field.)
- **The card means one thing:** "this DID stands behind this object". Two cards over the same `(did, root)` are equivalent. Whether a tree is the object's *current* version is answered by the tree's own `id` and `updated` (§12) and by mutable references, never by the card; retraction is a new version, not an un-signing. Replacing a card never touches content.
- **Intent comes from the format, not the card.** What "standing behind" a given object amounts to is defined by the format the object declares in its own `index.json` (`post/1.0`: the signer publishes this post). This is why the signing domain is objects, not trees: a card over a tree that is not a well-formed object (§8 format layer) is a signature without a meaning, and readers MUST treat such a signed tree as malformed rather than as a verified folder. **This format defines no signature over bare bytes.**
- Everything else that might be called intent belongs to another layer: who *sent* an object is the transport's (an authenticated DIDComm envelope); passing on someone's object is their card under one's own envelope; endorsing, replying to or quoting an object is a **new object** that refers to it.
- The card is an optional layer. Pairwise authenticated encryption already guarantees origin; a card is needed only for forwarding and third-party verification.
- Version 1 admits a single card: a signed object carries at most the one `card.jws`.

## 7. Transport and Packaging

- **The unit of transport is the object with its card.** When no card is involved, the bare object folder MAY travel as-is. Zipping a signed object (or a bare object) yields the self-contained file projection — "a fact is a mapping; any faithful container is legal" — so no separate zip profile is needed.
- Receiver pipeline: reconstruct the mapping → recognize signed object or bare object → validate per §8 → if a card is present, verify it. A card that fails verification degrades the projection to *unverified*; the object itself MUST still be accepted.
- The DIDComm share message (`object-share/1.0`, specified with `@estoc/agent-core`) does not carry the layout at all: it carries the card in its body and the blocks of the object's canonical tree — the manifest, and the raw block of each file — as attachments named by CID; the receiver verifies the tree from the root and reads the object out of it. The CID's codec decides how a block is read — a drisl CID as the manifest, a raw CID as a file — and an attachment's media type is never consulted. The message body does not restate metadata. A closure travels as one file as a [DASL CAR](https://dasl.ing/car.html) — CARv1 whose every block is named by exactly the 36 bytes of a DASL CID; a block named otherwise is dropped like one that fails its hash — with the header `{version: 1, roots: [root]}` and then the manifest and the leaves.

## 8. Malformed (One Judgment per Layer)

1. **Container layer**: no unique path → bytes mapping can be reproduced — a symbolic link, a duplicate zip entry, a name that is not UTF-8 or holds U+0000, a path that is also a directory of another (§1). A set of blocks is a container too, and its reproduction is judged the same way; what the blocks say is judged next.
2. **Format layer**: (object) no `index.json` at the root, or `index.json` not well-formed — not JSON / missing `format` / missing `id` / `content.path` escaping `files/` or naming a hidden entry; (hashed tree) a manifest block that is not the canonical form of §2.1 — other bytes for its value, another member, a `src` that is not a raw DASL CID in canonical spelling, a key that is not `/` plus a path a tree can hold, one `src` under two sizes, more than 1 MiB — or whose keys are not exactly a canonical tree (`/index.json` and paths under `/files/`, no hidden segment, nothing else), or that lacks `/index.json`, or that states a `size` a present leaf does not have: no conforming hasher computed such a root over any fact, so it is no object's identity; (signed object) `object/index.json` absent or not well-formed; (card) a JWS that is not an `estoc/object-card`, or whose `kid` is not the payload's `did`, or whose payload is not the one text of §6, or whose `root` is not the canonical string of a drisl DASL CID.
3. **Closure layer**: `content.path` names a path that is not in the mapping. A leaf the manifest names but the blocks at hand lack is not a hole in the object but a mapping not yet fully reproduced — a partial object, a state of transport (`object-share/1.0`), every path and size known; the closure judgment is made over the whole mapping.

NOT malformed: broken body links; a card that fails verification (degrade); unknown top-level members of `index.json`; entries beside `object/` and `card.jws` in a container (litter — in a *manifest* they are a format-layer defect, §2.1); an unknown `format` (generic processing); a leaf the reader does not hold or declined to fetch (a partial or unverifiable object — a state of the reader, not a defect of the object).

## 9. Example: a Post

The vocabulary contract of the type format `https://estoc.dev/post/1.0` is defined elsewhere; this section only illustrates structure-layer shape (mirrored in [examples/](examples/)).

```
sea-day/                      # folder name carries no semantics
├── index.json
└── files/
    ├── body.md
    └── images/
        └── sunset.png
```

```json
{
  "format": "https://estoc.dev/post/1.0",
  "id": "01a03110-7c1e-7b3a-9f42-3d5e8a1b2c04",
  "title": "A Day at the Sea",
  "published": "2026-08-24T10:30:00Z",
  "updated": "2026-08-24T10:30:00Z",
  "content": { "mediaType": "text/markdown", "path": "files/body.md" }
}
```

Body, `files/body.md`:

```
The tide started rising at dusk, light flat on the water.

![sunset](images/sunset.png)
```

Its manifest has three entries — `/index.json`, `/files/body.md`, `/files/images/sunset.png`, in that (DRISL) order — and the root of [examples/sea-day](examples/sea-day/), byte for byte, is `bafyreicdsejj526l225wrfl5cpxcgehq4pzbpxphocvmiuvy6dpwi467aa`.

A micro-post (the degenerate single-file form):

```json
{
  "format": "https://estoc.dev/post/1.0",
  "id": "01a03118-52aa-7c31-a0f4-8d2e91c7b355",
  "published": "2026-08-24T12:01:00Z",
  "content": { "mediaType": "text/markdown", "text": "Just saw a double rainbow." }
}
```

A lone `index.json` is a one-entry manifest; [examples/minimal](examples/minimal/) roots at `bafyreighoyuo2t5ymwyezn2uuzuxyamaqzgmdneypefczqchzibnfzt3v4`.

## 10. Renderer Contract (Informative)

```ts
interface EstocObject {
  meta: IndexJson                                  // index.json, verbatim
  read(path: string): Promise<Uint8Array>          // files/ subtree only
}
```

- The host reads signed objects and verifies cards; the renderer is a pure function with no network and no filesystem. The capability boundary is the canonical tree: `read` cannot escape `files/`.
- The dispatch key is `format` (unknown format → generic card renderer).
- Four inherited properties: projection-source agnosticism (vault, message, and zip converge on the same object value); privacy as a structural guarantee (the renderer has no ability to issue requests; CSP is demoted to a second line of defense); object values serialize across a sandboxed iframe; recursive dispatch by `format` (once the dependency table lands).

## 11. Future Versions (Reserved Constructs)

Later versions introduce, per the full design (each arriving as a version bump of the type format URIs):

- the `objects` dependency table (an import map): **immutable references** `{hash, id?}` — a value; the hash (a manifest CID in its canonical string form, §1 — a plain string, as the card's `root` is; DASL's `{"$link": …}` is for converting DRISL to JSON, which this format never does) is the sole trust anchor; locations never enter the index and are always deferred to transport — and **mutable references** `{id, links?}` — an entity; the anchor is identity, not location. The discriminator is the presence of `hash`; both present = malformed;
- accompanying dependencies `objects/<name>/` beside a signed object (inlining is necessarily immutable; entries are themselves signed objects, recursively) and `links.json` deferred-location hints (DASL's RASL — `rasl://<cid>/?hint=<host>`, fetched at `/.well-known/rasl/<cid>` — is a ready shape for a hint to a whole block; a RASL URL never addresses a path inside an object);
- the renderer's `resolve(name)` and recursive dispatch;
- **promotion**: wrapping `files/` bytes into an object of their own, trading identity for external referenceability (value → entity). Version 1 has no external references, so promotion is meaningless and undefined here.

Version-1 reader behavior toward all of these is already pinned (ignore / broken link), so the upgrade cannot break old readers.

## 12. Open Questions

- `updated` currently belongs to the vocabulary layer, but "latest-authenticated-wins on `id` + `updated`" is a version-arbitration rule generic to all entities — it may belong in the structure layer next to `id`. To be settled together with the dependency table (mutable references fetch the latest version).
- Where and in what style the vocabulary contracts (post/1.0: `content` required, `title` optional, …) are written down.
- **Large files.** This version hashes a file whole and lets a reader decline what it cannot hold (§2.1). A chunk list under `src` (the DASL CID document leaves room for one) or BDASL's streaming verification would be a breaking change to the closed manifest — a new structure specification, not a version of this one — deliberately deferred until an object that needs it exists; until then a big object travels by the package road and is verified by readers that can hold its leaves.
- **Per-file media types.** A MASL entry may carry `content-type`; this version keeps the manifest to `src` and `size` so that the index stays the only place a media type is declared. A projection that needs one per file (a tile, §10) derives it from the extension.

## Appendix A — From UnixFS to DASL (Informative)

The 2026-08-24 draft hashed the tree as UnixFS (`unixfs-v1-2025`). What changed, and why:

| | UnixFS draft | This draft |
|---|---|---|
| Tree encoding | dag-pb directory nodes, one per directory; HAMT shards past 256 KiB | one DRISL manifest for the whole mapping |
| File encoding | raw block up to 1 MiB, else a dag-pb node over 1 MiB raw chunks | one raw block, whatever the size |
| Root | dag-pb CID of the root directory node (`bafybei…`) | drisl CID of the manifest (`bafyrei…`) |
| Empty directory | expressible; forbidden by rule | not expressible |
| Hidden entries | the profile's default, restated | this format's rule |
| Canonical order | a rule on directory links, checked on read | DRISL's own, checked on read |
| Blocks to verify the shape | one per directory (plus shards, plus chunk indexes) | one |
| Fetches to resolve a path | depth + chunks | two |
| Sizes | `Tsize` on every link | `size` on every entry |
| Bounds | a raw block is at most 1 MiB | the manifest is at most 1 MiB; a leaf is bounded by the reader |
| Litter in a hashed tree | walked, then filtered out of the object | refused: no object hashes to such a root |
| Interop | `ipfs add -r` reproduces the root; kubo verifies | any DASL implementation reproduces the root; kubo stores and fetches the blocks but does not reproduce it |
| Reference implementation | importer/exporter, dag-pb, UnixFS, multiformats (≈ 4 MB of dependencies; 155 kB bundled) | ≈ 700 lines, no dependency beyond WebCrypto (15 kB bundled) |

The mapping, the object, the card, the signed object, the renderer contract and every reserved name are untouched: the hash encoding was always a function of the mapping, and this replaces the function.

**What a switch touches.** Downstream, the code that reads blocks: the object-share protocol's skeleton (one block now — the manifest — and the `index.json` leaf; roughly ten thousand entries fit the inline budget), a vault's block store (no 1 MiB bound on a raw block; a drisl block is sound when it is one canonical DRISL document whose links are DASL CIDs — the *manifest* shape is this format's judgment, not the store's, so that other DRISL blocks are not damage), the CID parser every layer shares (it must accept 0x71 before anything else moves), and the CLI's roots. One encoder and one strict decoder must serve every reader, or the same tree roots differently in two places.

**Migration.** Every root produced under the UnixFS draft — `bafybei…`, codec 0x70 — is a pre-DASL identity that nothing in this draft computes: a card over one is malformed here, and a block store built for this draft does not hold a dag-pb block. A switch therefore either reads both encodings for a while, selected strictly by the root's codec (0x70 the UnixFS reader, 0x71 the manifest reader, never mixed in one walk) with a written sunset, or bumps the vault folder version and re-signs what exists — today one www post and development vaults. This document only says that the two identities are unrelated: the same mapping has one root under each.
