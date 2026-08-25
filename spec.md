# Folder Object Format 1.0 — Structure Layer

**Status:** Draft — 2026-08-24. Not frozen, not implemented. Subject to incompatible change.

## Abstract

This document defines a content format whose only substrate is a plain file tree. An **object** is a tree of the shape `{index.json, files/}`: a pure fact with no container conventions and no exclusion rules. An object's version identity is the content hash of its canonical tree; its entity identity is a UUID carried in the index. Signing happens outside the tree: a **signed object** pairs the object with a detached card that means one thing — this DID stands behind this object. Signatures cover the logical tree, never container bytes.

## Scope

Version 1 of this specification covers **internal files only**: an object's own index and its own bytes.

It does not cover cross-object references — the `objects` dependency table (immutable and mutable references, inlined dependencies, deferred location hints). These are introduced in a later version of the format family; the names they will occupy are reserved by this document (§3.2, §5).

Also out of scope: the vocabulary contracts of individual types (e.g. the field requirements of a post — defined per type format, see §3.1), the choice of body markup language, and the exact shape of the share message that carries objects over DIDComm.

## 1. Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals.

**Fact.** A fact is a mapping from relative paths to byte strings — nothing more. It carries no timestamps, no permissions, no empty directories, no symbolic links. Directories exist only as prefixes of file paths: a container that holds an empty directory reproduces the same mapping as one that does not, so an empty directory is never part of an object and never enters its hash (the hash encoding below *can* express one; this format simply never produces one — and a receiver MUST treat a directory node with no entries in an object tree as a format-layer defect, §8).

**Container.** Any encoding from which the path → bytes mapping can be reproduced (a directory on disk, a zip file, a set of DIDComm attachments) is a legal container. **No container has normative status**; all semantics attach to the mapping.

**Hash encoding.** Hashes use the `signed-dir` tree-hash scheme, which is UnixFS under the IPIP-499 `unixfs-v1-2025` profile (CIDv1, sha-256, raw leaves, 1 MiB chunks, balanced layout, dag-pb directory nodes, HAMT sharding past 256 KiB of block bytes): a file of at most 1 MiB is the raw-codec CID of its bytes; a larger file roots in a dag-pb node over raw chunks; a directory is a dag-pb UnixFS directory node. One hash encoding serves the whole stack, and the same mapping hashes to the same root as `ipfs add`, so any UnixFS implementation can recompute or verify an object's version identity. The codec alone does not tell a file from a directory (a dag-pb CID can root either); the UnixFS `Data` field does.

## 2. Object Format

An object is a tree with exactly two things at its root:

```
<object>/
  index.json        # the object's index (§3)
  files/            # the object's own bytes (§4)
```

**Canonical tree.** The canonical tree of an object is `index.json` plus the entire `files/` subtree. It is taken by enumeration — never by name-based exclusion. There are no out-of-tree entries and no exclusion rules inside an object: **an object is pure fact**. The card stands beside the object (§5), not inside it.

**Identity.**

- **Version identity** = the root CID of the canonical tree (value semantics).
- **Entity identity** = the `id` field of the index (UUIDv7; stable across versions).

Entries outside the canonical tree that happen to share a container with an object (drafts in a vault, editor litter) are legal but are not part of the object.

**Identity is packaging-neutral** (core invariant). Whether a card is present, and how the object travels, are decided entirely outside the object's folder; the root CID is structurally unaffected. Packaging is a transport decision, never a semantic one.

## 3. index.json

```json
{
  "format": "https://estoc.dev/post/1.0",
  "id": "01a03110-7c1e-7b3a-9f42-3d5e8a1b2c04",
  "content": { "mediaType": "text/x-djot", "path": "files/body.dj" }
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

A type's own vocabulary members (`name`, `published`, `updated`, `summary`, `tag`, … — vocabulary borrowed from Activity Streams 2.0) are orthogonal to the structural members and live in the same `index.json`, as defined by that type's format document.

### 3.2 Generic processing and forward compatibility

- **Generic machinery does not read the value of `format`.** Hashing, closure computation, bundling, signing, and transport are all shape-driven: the canonical tree is taken by enumeration and the closure check reads only `content.path`. An object with an unknown `format` MUST still be storable, transportable, and verifiable; rendering falls back to a generic card.
- Readers MUST ignore unknown top-level members of `index.json`. (Vocabulary members and future structural versions layer on here.)
- `objects` is a **reserved name**. Version-1 writers MUST NOT emit it; version-1 readers MUST NOT interpret it. A future object carrying a dependency table therefore remains a well-formed object to a version-1 reader — its dependencies are simply invisible, and body references to them degrade to broken links.

## 4. `files/` — the Object's Own Bytes

- The entire `files/` subtree is the object's own bytes: **all of it enters the root hash, all of it is covered by the card, and none of it is declared per file.** Closure computation depends on no body parser: the canonical tree is itself the complete closure.
- Bytes in `files/` are **values**: they have no `id` and no independent life; their identity is absorbed by the object's root hash.
- Version 1 defines only the trivial encoding: paths inside `files/` are real paths, and directory structure carries no semantics (layout is the object's private business).
- **Body references** are written as in-tree relative paths: `![figure](files/images/fig1.jpg)`. A reference to a path that does not exist is a **broken link** — a rendering-layer placeholder, not a malformed object. An absolute `http(s)` URL in the body is an ordinary external link and MUST NOT be loaded automatically (under end-to-end encryption, an external fetch leaks the reader's identity); it opens on click.
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

- The card is a compact JWS, `alg: EdDSA`, `typ: estoc/object-card`, `kid` naming the signer's verification method, over the payload `{did, root}`, where `did` is a `did:key` and `root` is the root CID of the object's canonical tree. A JWS without that `typ` is not a card; a `kid` that is not the payload's `did` makes the card malformed.
- **The card means one thing:** "this DID stands behind this object". It is out-of-tree testimony about a fact and **nothing more**: no issue order, no expiry, no takedown form. Two cards over the same `(did, root)` are equivalent. Whether a tree is the object's *current* version is answered by the tree's own `id` and `updated` (§12) and by mutable references, never by the signature; retraction is a new version, not an un-signing. Replacing a card never touches content.
- **Intent comes from the format, not the card.** What "standing behind" a given object amounts to is defined by the format the object declares in its own `index.json` (`post/1.0`: the signer publishes this post). This is why the signing domain is objects, not trees: a card over a tree that is not a well-formed object (§8 format layer) is a signature without a meaning, and readers MUST treat such a signed tree as malformed rather than as a verified folder. **This format defines no signature over bare bytes.**
- Everything else that might be called intent belongs to another layer: who *sent* an object is the transport's (an authenticated DIDComm envelope); passing on someone's object is their card under one's own envelope; endorsing, replying to or quoting an object is a **new object** that refers to it.
- The card is an optional layer. Pairwise authenticated encryption already guarantees origin; a card is needed only for forwarding and third-party verification.
- Version 1 admits a single card: a signed object carries at most the one `card.jws`.

## 7. Transport and Packaging

- **The unit of transport is the object with its card.** When no card is involved, the bare object folder MAY travel as-is. Zipping a signed object (or a bare object) yields the self-contained file projection — "a fact is a mapping; any faithful container is legal" — so no separate zip profile is needed.
- Receiver pipeline: reconstruct the mapping → recognize signed object or bare object → validate per §8 → if a card is present, verify it. A card that fails verification degrades the projection to *unverified*; the object itself MUST still be accepted.
- The DIDComm share message (`object-share/1.0`, specified with `@estoc/agent-core`) does not carry the layout at all: it carries the card in its body and the UnixFS blocks of the object's canonical tree as attachments named by CID; the receiver verifies the tree from the root and reads the object out of it. The message body does not restate metadata.

## 8. Malformed (One Judgment per Layer)

1. **Container layer**: no unique path → bytes mapping can be reproduced.
2. **Format layer**: (object) no `index.json` at the root, or `index.json` not well-formed — not JSON / missing `format` / missing `id` / `content.path` escaping `files/`; a hashed object tree containing an empty directory node (a fact cannot contain one, so a signed root that reaches one was not computed over a fact); (signed object) `object/index.json` absent or not well-formed; (card) a JWS that is not an `estoc/object-card`, or whose `kid` is not the payload's `did`.
3. **Closure layer**: `content.path` names a path with no bytes in the mapping. This is the only possible hole in version 1 — `files/` is declaration-free, so "declared but absent" cannot arise.

NOT malformed: broken body links; a card that fails verification (degrade); unknown top-level members; entries beside `object/` and `card.jws`; an unknown `format` (generic processing).

## 9. Example: a Post

The vocabulary contract of the type format `https://estoc.dev/post/1.0` is defined elsewhere; this section only illustrates structure-layer shape (mirrored in [examples/](examples/)).

```
sea-day/                      # folder name carries no semantics
├── index.json
└── files/
    ├── body.dj
    └── images/
        └── sunset.png
```

```json
{
  "format": "https://estoc.dev/post/1.0",
  "id": "01a03110-7c1e-7b3a-9f42-3d5e8a1b2c04",
  "name": "A Day at the Sea",
  "published": "2026-08-24T10:30:00Z",
  "updated": "2026-08-24T10:30:00Z",
  "content": { "mediaType": "text/x-djot", "path": "files/body.dj" }
}
```

Body, `files/body.dj`:

```
The tide started rising at dusk, light flat on the water.

![sunset](files/images/sunset.png)
```

A micro-post (the degenerate single-file form):

```json
{
  "format": "https://estoc.dev/post/1.0",
  "id": "01a03118-52aa-7c31-a0f4-8d2e91c7b355",
  "published": "2026-08-24T12:01:00Z",
  "content": { "mediaType": "text/x-djot", "text": "Just saw a double rainbow." }
}
```

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

- the `objects` dependency table (an import map): **immutable references** `{hash, id?}` — a value; the hash is the sole trust anchor; locations never enter the index and are always deferred to transport — and **mutable references** `{id, links?}` — an entity; the anchor is identity, not location. The discriminator is the presence of `hash`; both present = malformed;
- accompanying dependencies `objects/<name>/` beside a signed object (inlining is necessarily immutable; entries are themselves signed objects, recursively) and `links.json` deferred-location hints;
- the renderer's `resolve(name)` and recursive dispatch;
- **promotion**: wrapping `files/` bytes into an object of their own, trading identity for external referenceability (value → entity). Version 1 has no external references, so promotion is meaningless and undefined here.

Version-1 reader behavior toward all of these is already pinned (ignore / broken link), so the upgrade cannot break old readers.

## 12. Open Questions

- `updated` currently belongs to the vocabulary layer, but "latest-authenticated-wins on `id` + `updated`" is a version-arbitration rule generic to all entities — it may belong in the structure layer next to `id`. To be settled together with the dependency table (mutable references fetch the latest version).
- Where and in what style the vocabulary contracts (post/1.0: `content` required, `name` optional, …) are written down.
