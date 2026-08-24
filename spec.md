# Folder Object Format 1.0 — Structure Layer

**Status:** Draft — 2026-08-24. Not frozen, not implemented. Subject to incompatible change.

## Abstract

This document defines a content format whose only substrate is a plain file tree. An **object** is a tree of the shape `{index.json, files/}`: a pure fact with no container conventions and no exclusion rules. An object's version identity is the content hash of its canonical tree; its entity identity is a UUID carried in the index. Signing and transport happen outside the tree, via a **bundle** that pairs the object with a detached signature card. Signatures cover the logical tree, never container bytes.

## Scope

Version 1 of this specification covers **internal files only**: an object's own index and its own bytes.

It does not cover cross-object references — the `objects` dependency table (immutable and mutable references, inlined dependencies, deferred location hints). These are introduced in a later version of the format family; the names they will occupy are reserved by this document (§3.2, §5).

Also out of scope: the vocabulary contracts of individual types (e.g. the field requirements of a post — defined per type format, see §3.1), the choice of body markup language, and the exact shape of the share message that carries bundles over DIDComm.

## 1. Conventions and Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [BCP 14](https://www.rfc-editor.org/info/bcp14) ([RFC 2119](https://www.rfc-editor.org/rfc/rfc2119), [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174)) when, and only when, they appear in all capitals.

**Fact.** A fact is a mapping from relative paths to byte strings — nothing more. It carries no timestamps, no permissions, no empty directories, no symbolic links.

**Container.** Any encoding from which the path → bytes mapping can be reproduced (a directory on disk, a zip file, a set of DIDComm attachments) is a legal container. **No container has normative status**; all semantics attach to the mapping.

**Hash encoding.** Hashes use the `signed-dir` tree-hash scheme: a leaf is the raw-codec CID of its bytes; a directory node is the CID of its dag-json node. One hash encoding serves the whole stack. The CID codec is a built-in discriminator: raw = bytes, dag-json = tree.

## 2. Object Format

An object is a tree with exactly two things at its root:

```
<object>/
  index.json        # the object's index (§3)
  files/            # the object's own bytes (§4)
```

**Canonical tree.** The canonical tree of an object is `index.json` plus the entire `files/` subtree. It is taken by enumeration — never by name-based exclusion. There are no out-of-tree entries and no exclusion rules inside an object: **an object is pure fact**. The signature card belongs to the bundle (§5), not to the object.

**Identity.**

- **Version identity** = the root CID of the canonical tree (value semantics).
- **Entity identity** = the `id` field of the index (UUIDv7; stable across versions).

Entries outside the canonical tree that happen to share a container with an object (drafts in a vault, editor litter) are legal but are not part of the object.

**Identity is packaging-neutral** (core invariant). Whether a card is present, and whether the object travels inside a bundle, are decided entirely outside the object's folder; the root CID is structurally unaffected. Packaging is a transport decision, never a semantic one.

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
  - **inline**: `{mediaType, text}` — in this form `files/` MAY be empty, so **a lone `index.json` is a complete object**.

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

## 5. Bundle Format

A bundle is the value of `bundle(object, card?)`:

```
<bundle>/
  object/           # the object's canonical tree, verbatim (object/index.json, object/files/…)
  card.jws          # signature card (optional)
```

- **Recognition.** A tree is a bundle if and only if `object/index.json` exists and is well-formed. There is no other container magic: format identity rests on in-tree shape.
- **Degenerate end.** `bundle(object)` is a bundle containing only `object/`; the inverse, unbundle, extracts `object/`. Bundle and unbundle are **identity-neutral** operations (§2 invariant, made structural).
- `objects/<name>/` (accompanying dependencies) and `links.json` (location hints) are **reserved entry names** of the bundle. Version-1 writers MUST NOT produce them; version-1 readers MUST ignore them (discard, without affecting the validity of the object itself).

## 6. Signing (an Optional Layer)

- The signature card is a `signed-dir` root card: a JWS over `{did, id, expires, root}`, where `root` is the root CID of the object's canonical tree. The card is out-of-tree testimony; **the verifiable projection is the bundle** — the materialization of `envelope(card, content)`. Replacing a card never touches content.
- The card is an optional layer. Pairwise authenticated encryption already guarantees origin; a card is needed only for forwarding and third-party verification.
- The signing domain is trees. **This format defines no signature over bare bytes.**
- Version 1 admits a single card: a bundle carries at most the one `card.jws`.

## 7. Transport and Packaging

- **The unit of transport is the bundle.** When no card is involved, the bare object folder MAY travel as-is. Zipping a bundle (or a bare object) yields the self-contained file projection — "a fact is a mapping; any faithful container is legal" — so no separate zip profile is needed.
- Receiver pipeline: reconstruct the mapping → recognize bundle or bare object → validate per §8 → if a card is present, verify it. A card that fails verification degrades the projection to *unverified*; the object itself MUST still be accepted.
- The DIDComm share message (specified elsewhere) carries the bundle's mapping entries as attachments (attachment `id` = relative path, e.g. `object/index.json`); the message body does not restate metadata.

## 8. Malformed (One Judgment per Layer)

1. **Container layer**: no unique path → bytes mapping can be reproduced.
2. **Format layer**: (object) no `index.json` at the root, or `index.json` not well-formed — not JSON / missing `format` / missing `id` / `content.path` escaping `files/`; (bundle) `object/index.json` absent or not well-formed.
3. **Closure layer**: `content.path` names a path with no bytes in the mapping. This is the only possible hole in version 1 — `files/` is declaration-free, so "declared but absent" cannot arise.

NOT malformed: broken body links; a card that fails verification (degrade); unknown top-level members; reserved bundle entries; an unknown `format` (generic processing).

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

- The host unwraps bundles and verifies cards; the renderer is a pure function with no network and no filesystem. The capability boundary is the canonical tree: `read` cannot escape `files/`.
- The dispatch key is `format` (unknown format → generic card renderer).
- Four inherited properties: projection-source agnosticism (vault, message, and zip converge on the same object value); privacy as a structural guarantee (the renderer has no ability to issue requests; CSP is demoted to a second line of defense); object values serialize across a sandboxed iframe; recursive dispatch by `format` (once the dependency table lands).

## 11. Future Versions (Reserved Constructs)

Later versions introduce, per the full design (each arriving as a version bump of the type format URIs):

- the `objects` dependency table (an import map): **immutable references** `{hash, id?}` — a value; the hash is the sole trust anchor; locations never enter the index and are always deferred to transport — and **mutable references** `{id, links?}` — an entity; the anchor is identity, not location. The discriminator is the presence of `hash`; both present = malformed;
- the bundle's accompanying dependencies `objects/<name>/` (inlining is necessarily immutable; entries are themselves bundles, recursively) and `links.json` deferred-location hints;
- the renderer's `resolve(name)` and recursive dispatch;
- **promotion**: wrapping `files/` bytes into an object of their own, trading identity for external referenceability (value → entity). Version 1 has no external references, so promotion is meaningless and undefined here.

Version-1 reader behavior toward all of these is already pinned (ignore / broken link), so the upgrade cannot break old readers.

## 12. Open Questions

- `updated` currently belongs to the vocabulary layer, but "latest-authenticated-wins on `id` + `updated`" is a version-arbitration rule generic to all entities — it may belong in the structure layer next to `id`. To be settled together with the dependency table (mutable references fetch the latest version).
- Where and in what style the vocabulary contracts (post/1.0: `content` required, `name` optional, …) are written down.
