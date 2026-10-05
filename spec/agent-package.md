# Agent Package Specification

> **Status:** Draft proposal for community review. This document is not an
> adopted standard, and provisional names may change.

## 1. Abstract

This document proposes a common format for packaging and distributing agent skills and agent plugins as Open Container Initiative (OCI) artifacts. It preserves their native file formats while adding an immutable distribution unit, typed discovery, a verifiable file inventory, and support for signatures and attestations through existing OCI tooling.

The proposal is intended for community feedback and interoperability work. It does not define a runnable container image or require a particular registry, marketplace, or agent host.

The proposal makes the concrete artifact visible at the OCI manifest boundary:

- A Skill uses `artifactType: application/vnd.agentpackage.skill.v1`.
- A Plugin uses `artifactType: application/vnd.agentpackage.plugin.v1`.

Skills and Plugins are distinct public artifact types, but use the same package-config media type and schema:

```text
application/vnd.agentpackage.config.v1+json
```

The common config provides a payload declaration, an external payload specification, a complete file inventory, per-file digests, extensions, and a consistent validation contract. Each v1 artifact is self-contained: a Plugin that includes Skills carries their files in its own payload.

This design makes Skills and Plugins easy to classify without fetching the config blob, while avoiding separate config schemas and separate installer implementations for artifacts whose packaging metadata is nearly identical.

## 2. Conformance

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **NOT RECOMMENDED**, **MAY**, and
**OPTIONAL** in this document are to be interpreted as described in BCP 14,
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and
[RFC 8174](https://www.rfc-editor.org/rfc/rfc8174), when, and only when, they
appear in all capitals.

Examples and explanatory rationale are non-normative unless explicitly stated
otherwise.

## 3. Goals

The proposal should provide:

- Manifest-level classification of Skills and Plugins.
- A shared config schema and media type for both artifact types.
- Immutable, versioned distribution through OCI registries.
- A complete inventory of the files carried directly by an artifact.
- Install-time integrity checks and support for authenticity verification.
- Compatibility with existing Skill and Plugin directory specifications.
- Marketplace approval through signatures and attestations attached as OCI referrers.
- Registry-independent artifact content: publishing the same artifact bytes to another registry does not require rebuilding its config.
- Local developer workflows alongside managed installation.

## 4. Non-Goals

This proposal does not attempt to standardize:

- The internal semantics already owned by the Agent Skills or Agent Plugins specifications.
- A universal runtime permission model.
- Host-specific install roots such as `.codex`, `.claude`, or `.agents`.
- Automatic installation of arbitrary operating-system or language dependencies.
- Arbitrary install scripts.
- Package encryption or entitlement-based content decryption.
- A global package namespace independent of registries.
- Inter-artifact dependencies or composition by reference. A possible future model is discussed in Appendix B.

## 5. Terminology

**Artifact**: an OCI Image Manifest plus its referenced config and content blobs.

**Skill artifact**: an artifact whose `artifactType` is `application/vnd.agentpackage.skill.v1` and whose payload is one Skill.

**Plugin artifact**: an artifact whose `artifactType` is `application/vnd.agentpackage.plugin.v1` and whose payload is one Plugin.

**Package Config**: the structured JSON config shared by Skill and Plugin artifacts. This proposal does not require a canonical JSON serialization; OCI digests identify the exact published bytes.

**Direct payload**: files stored in the artifact's own content layer.

**Materialized tree**: the complete directory view presented to a host from the artifact payload. Materialization does not require copying payload files into a writable directory; an implementation may provide an equivalent view through a read-only mount, projection, or content-addressed store.

**Payload specification**: the external specification governing the materialized payload's internal structure and semantics.

**Artifact digest**: the immutable OCI digest of the artifact manifest.

**Installer**: the client component that pulls, verifies, materializes, and inventories artifacts, and applies local installation policy. It may run in a desktop application, command-line tool, service, or agent host.

## 6. OCI Profiles

This proposal uses the standard OCI Image Manifest media type:

```text
application/vnd.oci.image.manifest.v1+json
```

The initial profiles are:

| OCI field | Skill artifact | Plugin artifact |
|---|---|---|
| `artifactType` | `application/vnd.agentpackage.skill.v1` | `application/vnd.agentpackage.plugin.v1` |
| `config.mediaType` | `application/vnd.agentpackage.config.v1+json` | `application/vnd.agentpackage.config.v1+json` |
| `layers[0].mediaType` | `application/vnd.agentpackage.skill.payload.v1.tar+gzip` | `application/vnd.agentpackage.plugin.payload.v1.tar+gzip` |
| `payload.kind` | `agent-skill` | `agent-plugin` |
| Direct payload | One Skill directory | One Plugin directory |

Each artifact MUST use the OCI Image Manifest structure, with `schemaVersion` set to `2` and `mediaType` set to `application/vnd.oci.image.manifest.v1+json`. It MUST contain exactly one Package Config descriptor and exactly one content layer, using the media types in the table above. The initial profiles do not define dependencies on other artifacts, layer ordering, overlay, or whiteout semantics. The content layer is a gzip-compressed tar archive of the payload, not an OCI filesystem changeset.

The artifact types and content-layer media types are concrete because the outer objects and payloads have different semantics. The config media type is common because the package metadata and validation model are shared.

The `application/vnd.agentpackage.*` names are provisional community-facing media types. Before a final specification, the project should establish a change controller and registration strategy in accordance with media-type naming requirements. The separation between Skill and Plugin artifact types does not depend on the final namespace choice.

### 6.1 Artifact And Payload Classification

`artifactType` enables classification without fetching the config. `payload.kind` enables one common config schema to be parsed and validated consistently after it is fetched.

The two values MUST agree:

```text
application/vnd.agentpackage.skill.v1  <-> agent-skill
application/vnd.agentpackage.plugin.v1 <-> agent-plugin
```

An installer MUST reject an artifact when the mapping does not match. This strict rule prevents ambiguity and makes corrupted or incorrectly assembled artifacts fail closed.

### 6.2 Distribution Identity

The OCI reference used to retrieve an artifact and the resolved artifact digest establish its distribution identity. Embedding the artifact's own registry repository and tag inside the config makes identical content change when copied between registries and creates conflicting sources of truth.

The Package Config has no self-identity object. Clients record `requestedRef`, `resolvedRef`, registry, repository, tag, and digest in local inventory.

## 7. OCI Manifest Annotations

The OCI manifest SHOULD use standard annotations for small display and provenance metadata:

```json
{
  "org.opencontainers.image.title": "Case Triage",
  "org.opencontainers.image.description": "Triage support cases using approved procedures",
  "org.opencontainers.image.version": "1.2.3",
  "org.opencontainers.image.source": "https://gitlab.example.com/team/case-triage",
  "org.opencontainers.image.revision": "abc123",
  "org.opencontainers.image.licenses": "Apache-2.0",
  "org.opencontainers.image.documentation": "https://example.org/case-triage/docs"
}
```

These annotations are for discovery and display. They are not substitutes for the Package Config, content digests, signatures, or attestations.

Source URL and source revision belong on the OCI manifest and MUST NOT be duplicated in the Package Config's `annotations` map.

When a standard OCI annotation repeats metadata defined by the native payload specification, the values MUST be consistent. For example, a Plugin payload version and `org.opencontainers.image.version` must match when both are present.

This proposal does not assign a normative custom annotation namespace for the payload ID. OCI annotation keys should use a namespace controlled by the specification owner, and that ownership is not yet settled. A future revision may define manifest-visible Skill and Plugin ID annotations. If an implementation projects an ID under its own namespace, it MUST exactly match `payload.id`.

`org.opencontainers.image.version` is descriptive metadata, not an immutable identity. Clients MUST resolve tags to artifact digests and use digests for verification and inventory.

## 8. Common Package Config

Both artifact types use:

```text
application/vnd.agentpackage.config.v1+json
```

The illustrative JSON shape is shown below. Values such as `string`, `URI`, and `agent-skill | agent-plugin` are placeholders, not literal field values or a machine-readable schema.

```json
{
  "schemaVersion": "agent.package/v1",
  "payload": {
    "id": "string",
    "kind": "agent-skill | agent-plugin",
    "spec": "URI",
    "specVersion": "string",
    "root": "payload"
  },
  "files": [
    {
      "path": "payload/relative/path",
      "digest": "sha256:<hex-encoded digest>",
      "executable": false
    }
  ],
  "annotations": {},
  "extensions": {}
}
```

### 8.1 Required Fields

The Package Config MUST be a JSON object. `payload`, `annotations`, and `extensions` MUST be objects; `files` MUST be an array of objects. `schemaVersion`, the five required payload fields listed in Section 8.2, and each file's `path` and `digest` MUST be present as nonempty strings. When present, `files[].executable` MUST be a boolean. Config annotation values MUST be strings. JSON objects in the Package Config MUST NOT contain duplicate member names.

`schemaVersion` MUST be `agent.package/v1`.

`payload` MUST identify the single materialized payload represented by the artifact.

`files` MUST list every regular file carried in the artifact's direct content layer. It MAY be empty only when the direct content layer contains no regular files.

`annotations` and `extensions` MUST be present and MAY be empty objects.

### 8.2 Payload

`payload.id` is the stable, package-local ID of the Skill or Plugin.

`payload.kind` is the shared-config discriminator and MUST agree with the OCI `artifactType`.

`payload.spec` MUST be an absolute URI identifying the external payload specification or its canonical schema.

`payload.specVersion` identifies the version or profile of that specification. The pair of `payload.spec` and `payload.specVersion` selects validation rules supported by the installer; a URI alone does not establish trust or require downloading a validator.

`payload.root` identifies the payload directory inside the archive and MUST be `payload` in the initial profiles. This archive prefix is not the installed directory name. An installer presents the contents of this directory as the native Skill or Plugin root and preserves any directory-naming requirements of the referenced payload specification. For example, a Skill named `case-triage` is presented in a directory named `case-triage`, with `SKILL.md` directly inside it.

For a Skill, `payload.id` MUST match the Skill name declared by the referenced Skill specification. For a Plugin, it MUST match the Plugin name declared by the referenced Plugin specification.

### 8.3 Files

Every regular file carried directly in `layers[0]` MUST appear exactly once in `files`. Every `files` entry MUST correspond to one regular file in the direct payload archive.

The OCI layer digest verifies the payload archive as transported. Each `files[].digest` verifies the bytes of one regular file after materialization. The file inventory therefore enables a consumer to identify changed files and to detect missing, unexpected, or duplicate files independently of the archive's compression and metadata representation.

When verifying an installed tree, the path after the `payload/` prefix is relative to the native installation root. For example, `payload/SKILL.md` maps to `case-triage/SKILL.md` in the Skill installation shown in Section 9.3.

For `agent.package/v1`, `digest` MUST use the form `sha256:<encoded>`, where `<encoded>` is exactly 64 lowercase hexadecimal characters containing the SHA-256 digest of the regular file's byte sequence. The digest does not include the file path, archive header, timestamps, ownership, permissions, or compression. The Package Config binds each path to its expected content, and its OCI descriptor binds the config to the artifact manifest.

File paths:

- MUST be relative to the artifact content root.
- MUST begin with `payload/` in the initial profiles.
- MUST use `/` as the separator.
- MUST NOT contain empty, `.` or `..` path components, backslashes, NUL characters, or a platform drive prefix.
- MUST NOT be absolute or resolve outside the payload root.

These rules also apply to effective archive entry paths after processing tar metadata. Directory entries MAY name `payload` itself and MAY have one trailing `/`; they are not listed in `files`. All other directory entries MUST be below `payload/`. An installer MUST reject duplicate entry paths, file/directory conflicts, and paths that cannot be represented distinctly and safely on its target filesystem, including collisions caused by case folding or platform-specific normalization.

Payload archives MUST NOT contain device files, sockets, FIFOs, hard links, or symlinks. A later profile may permit constrained symlinks if there is demonstrated need.

The optional `executable` field records whether any POSIX executable bit is set in the archive entry. It defaults to `false` when omitted. A builder MUST set it to `true` when any executable bit is set, and an installer MUST reject an archive entry whose mode does not match the declared value. Installers MUST NOT execute payload files during installation.

Archive ownership and other permission bits do not authorize privilege changes. Installers MUST NOT restore set-user-ID or set-group-ID bits from the archive. They apply local policy to ownership and other permissions; platforms without POSIX executable bits retain the declared value as metadata.

### 8.4 Package Config Annotations

The config-level `annotations` map is reserved for namespaced package metadata that is not intended for manifest-only discovery.

It MUST NOT redefine or override required fields, file integrity, `artifactType`, or the standard OCI manifest annotations listed in Section 7.

### 8.5 Extensions

`extensions` contains namespaced experimental or profile-specific structured metadata.

Example:

```json
{
  "extensions": {
    "com.example.requirements.v1": {
      "tools": ["sfdc-cli"],
      "network": ["https://salesforce.example.com"]
    }
  }
}
```

Extensions MUST NOT change the meaning of required fields.

## 9. Skill Artifact Example

The following artifact packages one Agent Skills-compatible `case-triage` Skill.

The examples in Sections 9 and 10 use placeholder digests and illustrative descriptor sizes. They are not complete, downloadable artifacts. Builders must calculate digests and sizes from the actual config and compressed payload bytes.

### 9.1 OCI Manifest

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/vnd.agentpackage.skill.v1",
  "config": {
    "mediaType": "application/vnd.agentpackage.config.v1+json",
    "digest": "sha256:1111111111111111111111111111111111111111111111111111111111111111",
    "size": 972
  },
  "layers": [
    {
      "mediaType": "application/vnd.agentpackage.skill.payload.v1.tar+gzip",
      "digest": "sha256:2222222222222222222222222222222222222222222222222222222222222222",
      "size": 18432
    }
  ],
  "annotations": {
    "org.opencontainers.image.title": "Case Triage",
    "org.opencontainers.image.description": "Triage support cases using approved procedures",
    "org.opencontainers.image.version": "1.2.3",
    "org.opencontainers.image.source": "https://gitlab.example.com/team/case-triage",
    "org.opencontainers.image.revision": "abc123",
    "org.opencontainers.image.licenses": "Apache-2.0"
  }
}
```

### 9.2 Shared Package Config

```json
{
  "schemaVersion": "agent.package/v1",
  "payload": {
    "id": "case-triage",
    "kind": "agent-skill",
    "spec": "https://agentskills.io/specification",
    "specVersion": "unversioned",
    "root": "payload"
  },
  "files": [
    {
      "path": "payload/SKILL.md",
      "digest": "sha256:0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
    },
    {
      "path": "payload/scripts/analyze.py",
      "digest": "sha256:abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123456789",
      "executable": false
    },
    {
      "path": "payload/references/policy.md",
      "digest": "sha256:123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0"
    }
  ],
  "annotations": {},
  "extensions": {}
}
```

The linked Agent Skills specification does not currently declare a numbered format version. `unversioned` is an illustrative profile label, not a version published by that project. An interoperability profile must agree on the supported revision and label before relying on this example for validation.

### 9.3 Direct And Materialized Layout

The archive contains:

```text
payload/
  SKILL.md
  scripts/
    analyze.py
  references/
    policy.md
```

The installer presents the same files under the native Skill directory name:

```text
case-triage/
  SKILL.md
  scripts/
    analyze.py
  references/
    policy.md
```

The Package Config is the OCI config blob. It is not copied into the content archive.

## 10. Self-Contained Plugin Artifact Example

This example packages one portable Plugin with a root `plugin.json`, two bundled Skills, MCP server configuration, and assets. The exact payload structure is governed by the referenced Plugin specification, not by agent-package.

### 10.1 OCI Manifest

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "artifactType": "application/vnd.agentpackage.plugin.v1",
  "config": {
    "mediaType": "application/vnd.agentpackage.config.v1+json",
    "digest": "sha256:3333333333333333333333333333333333333333333333333333333333333333",
    "size": 1544
  },
  "layers": [
    {
      "mediaType": "application/vnd.agentpackage.plugin.payload.v1.tar+gzip",
      "digest": "sha256:4444444444444444444444444444444444444444444444444444444444444444",
      "size": 32768
    }
  ],
  "annotations": {
    "org.opencontainers.image.title": "Support Operations",
    "org.opencontainers.image.description": "Support workflows and service connections",
    "org.opencontainers.image.version": "0.3.0",
    "org.opencontainers.image.source": "https://gitlab.example.com/team/support-operations",
    "org.opencontainers.image.revision": "def456",
    "org.opencontainers.image.licenses": "Apache-2.0"
  }
}
```

### 10.2 Shared Package Config

```json
{
  "schemaVersion": "agent.package/v1",
  "payload": {
    "id": "support-operations",
    "kind": "agent-plugin",
    "spec": "https://agent-plugins.org/schemas/1.0.0/plugin.schema.json",
    "specVersion": "1.0.0",
    "root": "payload"
  },
  "files": [
    {
      "path": "payload/plugin.json",
      "digest": "sha256:23456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01"
    },
    {
      "path": "payload/skills/case-triage/SKILL.md",
      "digest": "sha256:3456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef012"
    },
    {
      "path": "payload/skills/customer-summary/SKILL.md",
      "digest": "sha256:456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef0123"
    },
    {
      "path": "payload/mcp.json",
      "digest": "sha256:56789abcdef0123456789abcdef0123456789abcdef0123456789abcdef01234"
    },
    {
      "path": "payload/assets/icon.svg",
      "digest": "sha256:6789abcdef0123456789abcdef0123456789abcdef0123456789abcdef012345"
    }
  ],
  "annotations": {},
  "extensions": {}
}
```

### 10.3 Direct And Materialized Layout

The content layer contains the complete Plugin payload under the archive prefix `payload/`. The installer presents its contents as the Plugin root, for example `support-operations/`:

```text
payload/
  plugin.json
  mcp.json
  skills/
    case-triage/
      SKILL.md
    customer-summary/
      SKILL.md
  assets/
    icon.svg
```

## 11. Commonality Between Skills And Plugins

The two artifact profiles share one package contract:

| Concern | Skill | Plugin | Shared rule |
|---|---|---|---|
| OCI transport | OCI Image Manifest | OCI Image Manifest | Same |
| Config descriptor | `application/vnd.agentpackage.config.v1+json` | `application/vnd.agentpackage.config.v1+json` | Same |
| Config schema | `agent.package/v1` | `agent.package/v1` | Same |
| Payload declaration | `payload` object | `payload` object | Same fields |
| File inventory | `files` | `files` | Same verification algorithm |
| Config annotations | `annotations` | `annotations` | Same constraints |
| Extensions | `extensions` | `extensions` | Same namespacing rules |
| Signatures and attestations | OCI referrers | OCI referrers | Same trust mechanism |
| Local inventory | Resolved digest and projection | Resolved digest and projection | Same base record |

They differ only where a consumer needs the distinction:

- The top-level `artifactType`.
- The matching `payload.kind` value.
- The content-layer media type.
- The external payload specification.

An installer can therefore implement one flow:

```text
fetch OCI manifest
  -> classify by artifactType
  -> fetch and parse the common Package Config
  -> verify artifactType/payload.kind agreement
  -> verify direct content against files[]
  -> validate the materialized tree against payload.spec
  -> project or mount it for the host
  -> record inventory
```

## 12. Validation And Installation

A conforming installer MUST:

1. Resolve the requested reference to an immutable manifest digest, preserving any digest supplied by the caller.
2. Fetch the manifest, verify its bytes against that digest, and require the OCI manifest structure and profile defined in Section 6.
3. Recognize or reject the manifest's `artifactType` according to local policy, and verify referenced blob digests and descriptor sizes before processing those blobs.
4. Require `config.mediaType` to be `application/vnd.agentpackage.config.v1+json` for this profile.
5. Validate the Package Config against `agent.package/v1`.
6. Reject an `artifactType` and `payload.kind` mismatch.
7. Require exactly one content layer and verify that its media type matches the artifact profile.
8. Verify every direct regular file against `files` before committing the installation.
9. Reject missing, extra, duplicate, or unsafe file paths.
10. Verify that each regular file archive entry's executable mode matches its `files` entry.
11. Select supported validation rules for `payload.spec` and `payload.specVersion`, reject unsupported pairs, and validate the native payload tree, including any directory-naming requirements.
12. Apply local signature, attestation, approval, and revocation policy to the artifact.
13. Record the requested reference, resolved artifact digest, verification results, and projection paths in local inventory.

Installers MUST NOT execute package-provided code during installation.

Installers MUST materialize archive entries within an isolated staging root or an equivalently protected store, without following filesystem links outside that root. They MUST apply resource limits during download and decompression, and MUST complete integrity, payload, and policy checks before making an installation available to the host. A failed check MUST NOT leave a partially verified installation active. Installers MUST NOT automatically fetch or execute validators from package-supplied specification URIs; validation uses rules the installer already trusts.

The content archive does not include the Package Config. A client that extracts files into a host directory retains origin and verification metadata in its inventory store. A client that mounts a content-addressed tree may keep the OCI config alongside its internal store, but it does not expose that file as part of the native Skill or Plugin payload unless the payload specification independently requires it.

## 13. Signatures, Attestations, And Policy

Signatures, evaluations, SBOMs, provenance, and marketplace approvals SHOULD be separate OCI artifacts whose `subject` descriptor identifies the immutable digest of the Skill or Plugin artifact.

The registry stores and exposes the referrer graph; it does not establish trust. Clients MUST verify signed statements and signer identities according to local policy.

File digests establish expected contents but do not establish publisher identity. A consumer whose policy requires authenticity MUST verify an applicable signature or other trusted statement over the artifact manifest. A signature over the manifest digest transitively binds the Package Config and direct content descriptors referenced by that manifest.

A recommended enterprise installation profile requires:

- An approved OCI registry.
- Immutable digest resolution.
- Valid package and file integrity.
- A marketplace approval attestation for the artifact.
- No revoked artifact digest.
- A successful payload-spec validation of the final materialized tree.
- A complete local inventory record.

## 14. Local Inventory

Inventory is an installer profile, not part of the portable Package Config.

Example:

```json
{
  "schemaVersion": "agent.package.install/v1",
  "installedAt": "2026-09-14T15:00:00Z",
  "mode": "managed",
  "requestedRef": "oci://registry.example.com/agents/support-operations:0.3.0",
  "resolvedRef": "oci://registry.example.com/agents/support-operations@sha256:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa",
  "artifactType": "application/vnd.agentpackage.plugin.v1",
  "payload": {
    "id": "support-operations",
    "kind": "agent-plugin",
    "projectedTo": ["/home/user/.agents/plugins/support-operations"]
  },
  "verification": {
    "integrity": "passed",
    "policy": "allowed"
  }
}
```

## 15. Security Considerations

- A verified signature authenticates a signed statement about an immutable digest; it does not establish that the statement is true or that the content is safe.
- Registry association is not trust. Clients must verify signer identity and signed statements.
- Installers must defend against archive traversal, path confusion, unexpected file types, duplicate paths, decompression bombs, and resource exhaustion.
- Installers must not execute package content during installation.
- The materialized tree must be validated against its payload specification.
- Local mutation invalidates previously verified installed-state integrity.

## 16. Open Questions

The core Skill/Plugin split and common config are defined by this proposal. The following details require community agreement:

1. Who should serve as change controller for the `application/vnd.agentpackage.*` media-type namespace, and should the media types be registered with IANA?
2. Should Skill and Plugin payload layers use type-specific media types, as proposed here, or share one generic agent-package payload media type?
3. What controlled annotation namespace should expose `payload.id` at the manifest level, if any?
4. Which Plugin specification URI and version negotiation rules should be normative for the first interoperability profile?
5. What limits should the base profile set for archive size, expanded size, and file count?
6. How should an interoperability profile identify and pin a revision of an unversioned payload specification, such as Agent Skills?
7. Should unknown Package Config fields be rejected or ignored, and how should clients handle extensions they do not understand? These rules need to be settled before claiming interoperable v1 implementations.
8. Should the project publish a machine-readable Package Config schema and complete test artifacts with reproducible digests?

## 17. References

- [OCI Image Manifest Specification](https://github.com/opencontainers/image-spec/blob/main/manifest.md)
- [OCI Content Descriptors](https://github.com/opencontainers/image-spec/blob/main/descriptor.md)
- [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)
- [OCI Annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md)
- [OpenAI: Package your plugin](https://developers.openai.com/plugins/build/plugins)
- [Agent Skills Specification](https://agentskills.io/specification)
- [Agent Plugins Specification](https://agent-plugins.org/specification)
- [Agent Plugins Manifest Schema, version 1.0.0](https://agent-plugins.org/schemas/1.0.0/plugin.schema.json)

## Appendix A. Post-Installation Integrity Verification (Non-Normative)

The normative validation procedure establishes that the materialized payload matches the published artifact at installation time. A client may also verify that relationship later, but the frequency, enforcement point, and response are deployment policy rather than requirements of this specification.

### A.1 Extracted Installations

A client that copies files into a conventional directory can recalculate the digests in `files` at startup, periodically, or on demand. It can also check for missing or unexpected regular files. The expected inventory should come from an authenticated Package Config retained outside the writable payload, or be retrieved through the immutable artifact digest; a mutable inventory stored beside the extracted files is not an independent integrity reference.

An implementation might report a changed installation as `managed-modified` and then warn, refuse activation, repair, quarantine, or reinstall it according to local policy. Runtime state, caches, logs, generated files, and user configuration are normally better stored outside the managed payload so intentional changes are not confused with package modification.

An independently signed file included in a native payload may provide another way to verify an extracted directory when the OCI artifact is unavailable. Its format and trust policy are outside this proposal. It is not a second Agent Package inventory. If present, the signature file is itself listed in `files` like any other direct regular file.

### A.2 Mounted Or Content-Addressed Installations

A consumer does not necessarily need to copy payload files into a writable directory. It can unpack the payload once into a content-addressed store and expose it through a read-only mount, bind mount, projection, or equivalent mechanism. Storage formats, filesystem adapters, or snapshotters may also support lazy or directly mounted artifact content.

A gzip-compressed tar layer is not itself a generally mountable filesystem, so the exact mechanism is implementation-specific. A mounted implementation still presents the same materialized tree and remains subject to the normative installation validation requirements.

Read-only, content-addressed storage can make modification less likely and can allow multiple installations to share the same underlying bytes. When the backing store and projection mechanism are trusted, repeated file-by-file startup checks may provide little additional value. Mutable runtime state should be placed in a separate writable location.

### A.3 Developer Mode And Local Mutation

Clients can distinguish among these installation states:

```text
managed
local-dev
managed-modified
```

Managed artifacts come from OCI artifacts verified under local installation policy, whether obtained from a publisher or a marketplace. Local-dev artifacts are loaded from a local authoring directory outside that managed installation flow; they may be signed or unsigned. Managed-modified artifacts began as managed installations but no longer match their verified file inventory.

A developer mode may load local-dev or managed-modified artifacts. Interfaces and inventory systems should distinguish them from policy-approved, integrity-verified content.

## Appendix B. Future Consideration: Composition By Reference (Non-Normative)

The v1 profiles are deliberately self-contained. A Plugin that includes a Skill carries that Skill's files in the Plugin content layer, and the Plugin's file inventory covers those files. This keeps installation, validation, signing, approval, revocation, and offline use centered on one artifact.

A future profile could allow a Plugin to place a separately published Skill artifact into its materialized tree. Such a profile might declare a digest-pinned reference and target path resembling:

```json
{
  "id": "case-triage",
  "kind": "agent-skill",
  "artifactType": "application/vnd.agentpackage.skill.v1",
  "ref": "oci://registry.example.com/agents/case-triage@sha256:bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb",
  "target": "payload/skills/case-triage"
}
```

Digest pinning would be essential. Publishing a new Skill would not change an existing Plugin: adopting the new Skill digest would also change the Plugin config and produce a new Plugin artifact digest. Mutable tags alone would not provide this property.

Composition by reference nevertheless introduces substantial design and security questions:

- whether every referenced artifact must be fetched from an authorized and available registry;
- how signatures, approvals, security evaluations, and revocations are applied independently to every component;
- whether the Plugin publisher assumes any responsibility for the referenced Skill;
- how installation remains atomic when one component cannot be fetched or verified;
- how offline installation, retention, garbage collection, and reproducible restoration work;
- how cycles, graph depth, path conflicts, and overlapping projections are rejected;
- how the completed tree is validated against the native Plugin specification; and
- whether a lock document, SBOM, or local inventory records the complete resolved graph.

A future proposal should demonstrate a compelling reuse or lifecycle benefit before adding this complexity. It should define composition in a new schema or profile rather than changing the meaning of the self-contained v1 config.
