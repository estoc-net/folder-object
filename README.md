# folder-object

**An object is a folder.** A content format whose only substrate is a plain file tree:

- An **object** is a tree of the shape `{index.json, files/}` — pure fact, no container magic, no exclusion rules.
- **Version identity = the root** — the [DASL](https://dasl.ing/) CID of the tree's **manifest**, one DRISL (deterministic CBOR) document mapping every path to the raw CID and size of its bytes, in the shape of a MASL bundle; files are hashed whole, there are no directory nodes (value semantics). **Entity identity = `id`** (UUIDv7, stable across versions).
- Signing happens outside the tree: a **signed object** is `{object/, card.jws}`, the card a JWS over `{did, root}` that means one thing — this DID stands behind this object, as the object's format defines it. Signatures cover the logical tree, never container bytes.
- Any container that can reproduce the path → bytes mapping (a directory, a zip file, a set of DIDComm attachments) is a legal carrier; no container has normative status.

Status: **Draft 1.0** (not frozen; the hash encoding is DASL, and the reference implementation — `@estoc/folder-object` 0.7, estoc branch `dasl` — implements this draft and reads nothing else). Version 1 covers internal files only; cross-object references (the `objects` dependency table) are introduced in a later version, with their names already reserved.

- Specification: [spec.md](spec.md)
- Examples: [examples/](examples/) — `sea-day/` is a post with an image; `minimal/` is the degenerate single-file form. The image is a 1×1 placeholder; it only demonstrates tree shape.
