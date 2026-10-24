# IntelliJ Dependency Source Mirror
## Design Document / Implementation Specification

**Version:** 1.0  
**Date:** 2026-10-24  
**Status:** Ready for Implementation  
**Target:** Existing IntelliJ IDEA Bridge Plugin + CLI  
**Primary OS:** macOS / APFS  
**Implementation Language:** Follow the existing plugin's Java/Kotlin stack  
**Consumers:** Codex CLI, ripgrep, file readers, other filesystem-based coding agents

---

# 1. Background

## 1.1 Existing Architecture

The existing system consists of:

1. An IntelliJ IDEA Bridge Plugin.
2. A CLI communicating with the plugin.
3. A Codex Skill describing CLI usage.
4. IntelliJ Platform PSI and indexing infrastructure.

The CLI allows Codex to access information about Java classes, methods, references and implementations through IntelliJ Platform APIs.

This works, but does not align with Codex's default tool-selection behavior.

Codex generally attempts native filesystem operations first:

```bash
rg "PaymentService"
rg --files | rg "PaymentProcessor"
sed -n '100,200p' src/main/java/.../PaymentService.java
```

When the target implementation is inside a Maven dependency, these searches frequently fail because the source resides in:

```text
~/.m2/repository/.../artifact-version-sources.jar
```

Codex must subsequently discover and read the bridge Skill, then call the bridge CLI.

This introduces unnecessary tool calls and repeated context acquisition.

## 1.2 Problem Statement

The objective is to make Maven dependency sources available to Codex as ordinary project-local files, without requiring Codex to explicitly invoke the bridge.

The solution must support:

- Multiple independent IntelliJ projects.
- Different dependency versions across projects.
- Large numbers of dependencies.
- Dependency source archives already attached to IDEA libraries.
- Incremental source updates.
- Maven SNAPSHOT dependencies.
- Multi-module projects.
- Native ripgrep and file-reading workflows.
- macOS APFS space-efficient storage.

## 1.3 Fundamental Design Principle

**Move dependency-source discovery out of the coding agent's execution path.**

The bridge is responsible for preparing and maintaining the filesystem representation before Codex needs it.

Codex should see:

```text
project/
├── pom.xml
├── src/
│   └── main/java/...
├── dependency-sources/
│   ├── com/company/core/1.2.0/...
│   └── com/company/payment/3.1.0/...
├── .gitignore
└── .rgignore
```

It should not need to know:

- Where Maven stores artifacts.
- How IDEA resolves dependencies.
- How a sources JAR is opened.
- Whether files are APFS clones.
- Whether the global cache has already been populated.
- How the IntelliJ Bridge CLI works.

The mirror is a read-only reference workspace, not another compilation source root.

---

# 2. Goals and Non-Goals

## 2.1 Functional Goals

**G1 — Native Search**

The following command, executed from the project root, must find mirrored dependency sources without special flags:

```bash
rg -n "class PaymentService"
```

**G2 — Native Read**

The following operations must work directly on ordinary filesystem paths:

```bash
cat dependency-sources/.../PaymentService.java
sed -n '100,200p' dependency-sources/.../PaymentService.java
```

**G3 — Version Correctness**

A project's mirror must represent the dependencies resolved by its own IDEA Project Model.

Different projects may mirror different versions of the same artifact.

**G4 — Global Cache Reuse**

Identical source archives must not require repeated extraction.

**G5 — Incremental Synchronization**

Unchanged source files should not be extracted or cloned again during normal synchronization.

**G6 — Transparent Lifecycle**

The plugin should synchronize automatically after project-model changes and restore a valid mirror after IDE restart.

**G7 — IDE Index Isolation**

The generated mirror must not be indexed as an additional set of project source files.

**G8 — Backward Compatibility**

Existing bridge functionality must continue working without behavioral changes.

## 2.2 Non-Goals

The initial release will not:

- Implement a FUSE filesystem.
- Change Maven resolution or dependency versions.
- Download sources independently through a custom Maven Resolver.
- Modify source archives in `~/.m2/repository`.
- Support writing changes back into source archives.
- Build a separate Java symbol index.
- Replace PSI-based semantic operations.
- Force Codex to use the bridge CLI.
- Mirror all artifacts in the user's Maven repository.
- Support seamless operation when the source JAR has never been attached or downloaded by IDEA.

The plugin must report missing sources rather than silently substituting an unrelated version.

---

# 3. Architecture

## 3.1 Components

Introduce the following logical components into the existing plugin:

```text
IntelliJ IDEA Plugin
│
├── Existing Bridge / CLI Infrastructure
│   └── Existing PSI capabilities
│
└── DependencySourceMirrorService
    │
    ├── DependencySnapshotProvider
    │   └── IntelliJ Project Model
    │
    ├── SourceArchiveResolver
    │   └── Library source roots / VFS
    │
    ├── MirrorScheduler
    │   └── Lifecycle, debounce, cancellation
    │
    ├── SourceCacheManager
    │   ├── Archive fingerprinting
    │   ├── Safe ZIP extraction
    │   ├── Content-addressed cache
    │   └── Cross-process locking
    │
    ├── ProjectMirrorSynchronizer
    │   ├── Incremental planning
    │   ├── APFS clone
    │   ├── Local mirror manifests
    │   └── Safe cleanup
    │
    ├── IdeExclusionManager
    │   └── Exclude generated mirror from IDEA indexing
    │
    ├── SearchVisibilityManager
    │   ├── .gitignore
    │   └── .rgignore
    │
    └── MirrorStatusService
        └── CLI status / diagnostics
```

## 3.2 Data Flow

```text
IDEA project opened
        │
        ▼
Wait for project-model readiness
        │
        ▼
Read IDEA Modules and Order Entries
        │
        ▼
Collect resolved Library source roots
        │
        ▼
Resolve archive paths and identifiers
        │
        ▼
Build DependencySnapshot
        │
        ▼
Compare with previous manifest
        │
        ▼
Hash changed / unknown archives
        │
        ▼
Populate global source cache
        │
        ▼
Create project-local APFS clones
        │
        ▼
Verify staging mirror
        │
        ▼
Publish mirror
        │
        ├── Update project manifest
        ├── Maintain Git / ripgrep visibility
        └── Ensure IDEA exclusion
        │
        ▼
Codex native rg / Read
```

## 3.3 Separation of Responsibilities

IDEA is authoritative for:

- Modules.
- Module dependencies.
- Attached Library source roots.
- Dependency accessibility within a module.
- Dependency changes.

The global cache is authoritative for:

- Extracted source content.
- Source archive fingerprints.
- Cache entry integrity.

The project mirror is authoritative for:

- Which ordinary source files Codex can currently search.
- Which source versions have been published.
- Which artifacts are unavailable.

No component may independently infer a newer dependency version from Maven Central or the local Maven repository.

---

# 4. Filesystem Layout

## 4.1 Global Cache

Default directory:

```text
~/Library/Caches/IntelliJDependencySourceMirror/
```

Proposed structure:

```text
IntelliJDependencySourceMirror/
├── objects/
│   ├── ab/
│   │   └── abcd.../
│   │       ├── source/
│   │       │   └── com/company/core/PaymentService.java
│   │       └── manifest.json
│   └── ef/
│       └── ef12.../
│           ├── source/
│           └── manifest.json
│
├── registry/
│   └── archive-registry.json
│
├── locks/
│   └── ...
│
└── tmp/
    └── ...
```

`objects/<prefix>/<sha256>/` identifies an extracted archive by its SHA-256 content hash.

The cache must not use GAV alone as the storage key.

Reason: SNAPSHOT artifacts and republished releases can reuse the same coordinates while containing different bytes.

## 4.2 Project Mirror

Default:

```text
<projectRoot>/dependency-sources/
```

Example:

```text
dependency-sources/
├── com/
│   └── company/
│       ├── core/
│       │   ├── 1.2.0/
│       │   │   └── com/company/core/PaymentService.java
│       │   └── 2.0.0/
│       │       └── com/company/core/PaymentService.java
│       └── payment/
│           └── 3.1.0/
│               └── com/company/payment/PaymentProcessor.java
└── ...
```

The layout preserves dependency identity and avoids overlapping files from unrelated artifacts.

GAV-derived paths are preferred when reliable metadata is available.

If an IDEA Library does not provide reliable Maven coordinates, use a deterministic fallback:

```text
dependency-sources/_libraries/<safe-library-name>-<identity-hash>/
```

The fallback identity must incorporate the normalized binary/source-root identity so that distinct libraries cannot collide.

Never invent Maven coordinates from an arbitrary filename.

## 4.3 Multi-Module Projects

A single project may contain modules with different versions of the same dependency.

Therefore the mirror should contain the union of resolved libraries, while preserving module membership in the manifest.

Example:

```text
module-a → core:1.2.0
module-b → core:2.0.0
```

Both versions appear in the mirror.

This is intentional.

Since `rg` is textual rather than module-aware, searches may return both versions.

The manifest must provide enough information to identify the relevant module version.

Future iterations may optionally support module-specific views:

```text
dependency-sources-by-module/module-a/...
```

Do not create these additional trees by default because they unnecessarily expand search results.

## 4.4 Metadata Location

Do not place operational metadata in a hidden subdirectory that must be recursively searched with ordinary `rg`.

Prefer:

```text
<projectRoot>/.idea/dependency-source-mirror/
    manifest.json
    sync-state.json
```

when the project uses a writable directory-based IDEA configuration.

Otherwise, keep metadata in the plugin's per-project state/cache area, keyed by a stable canonical project path and project identity.

The implementation must not assume every project uses directory-based `.idea` storage.

---

# 5. IntelliJ Project Model Integration

## 5.1 Dependency Enumeration

Implement:

```java
interface DependencySnapshotProvider {
    DependencySnapshot capture(Project project);
}
```

The implementation should inspect:

```java
ModuleManager.getInstance(project).getModules()
```

For each module:

```java
ModuleRootManager.getInstance(module)
```

Use the module's ordered dependencies to identify libraries.

Relevant IntelliJ concepts include:

- `OrderEnumerator`
- `OrderEntry`
- `LibraryOrderEntry`
- `Library`
- `OrderRootType.SOURCES`
- `ModuleOrderEntry`
- `ModuleRootManager`

Do not derive dependencies by parsing `pom.xml`.

The Project Model is already the imported and resolved model.

## 5.2 Dependency Scope

The provider must distinguish:

- Direct libraries.
- Libraries reachable through module dependencies.
- Compile dependencies.
- Test dependencies.
- Runtime dependencies.
- SDK entries.

The default inclusion policy should be:

```text
production libraries: included
test libraries: included
runtime libraries: included
project SDK/JDK: excluded
project-owned source roots: excluded
```

The final library collection should be deduplicated by source-root identity plus library identity.

Do not assume a library referenced by several modules should be extracted several times.

The concrete traversal API must be selected against the project's supported IntelliJ Platform SDK version.

## 5.3 Avoiding Source Mirror Feedback

The mirror must never become an input dependency.

Exclude it before or during initial materialization.

Enumeration must explicitly ignore paths inside the mirror directory, even if an exclusion operation has not yet completed.

Otherwise, plugin-generated files could accidentally enter later snapshots.

## 5.4 Source Root Types

Support at least:

1. JAR source root.
2. Ordinary directory source root.
3. Multiple source roots attached to one library.
4. Missing source root.
5. Unreadable source root.

For JAR sources:

```text
jar:///.../artifact-sources.jar!/
```

Resolve the backing local archive when possible.

For ordinary directory source roots, the default behavior should be to skip materialization and report them as externally accessible directories.

An optional future mode can clone directory-based source roots, with explicit filtering of VCS metadata and build outputs.

## 5.5 Source Lookup Limitations

A successful dependency import does not guarantee that every library has a source archive attached.

The provider must distinguish:

```text
AVAILABLE
MISSING
UNSUPPORTED_ROOT
UNREADABLE
```

The mirror service must not silently download sources or substitute decompiled class files in version 1.

---

# 6. Data Models

Use strongly typed internal models.

## 6.1 DependencySnapshot

```java
record DependencySnapshot(
    String projectId,
    String projectPath,
    long capturedAtEpochMillis,
    List<ModuleSnapshot> modules,
    List<LibrarySnapshot> libraries
) {}
```

## 6.2 ModuleSnapshot

```java
record ModuleSnapshot(
    String moduleId,
    String moduleName,
    List<String> libraryIds
) {}
```

## 6.3 LibrarySnapshot

```java
record LibrarySnapshot(
    String libraryId,
    String displayName,
    String groupId,
    String artifactId,
    String version,
    List<SourceRootSnapshot> sourceRoots,
    List<String> usedByModules
) {}
```

Coordinates must be nullable or represented by an optional metadata type.

## 6.4 SourceRootSnapshot

```java
record SourceRootSnapshot(
    String rootId,
    String vfsUrl,
    String localArchivePath,
    SourceRootKind kind,
    SourceAvailability availability
) {}
```

## 6.5 CacheEntry

```java
record CacheEntry(
    String sha256,
    String extractionFormatVersion,
    long extractedAt,
    long fileCount,
    long totalUncompressedBytes,
    String cachePath
) {}
```

## 6.6 Project Manifest

Example:

```json
{
  "schemaVersion": 1,
  "projectId": "project-a",
  "snapshotFingerprint": "abc123...",
  "generatedAt": "2026-10-24T12:00:00Z",
  "artifacts": [
    {
      "libraryId": "lib-001",
      "coordinates": "com.company:core:1.2.0",
      "sourceSha256": "012345...",
      "mirrorPath": "com/company/core/1.2.0",
      "modules": [
        "payment-service",
        "order-service"
      ],
      "status": "READY"
    }
  ]
}
```

Additional status values:

```text
READY
MISSING_SOURCES
EXTRACTION_FAILED
MATERIALIZATION_FAILED
UNSUPPORTED_ROOT
```

A manifest must be written atomically and must correspond to the published mirror.

---

# 7. Global Cache Design

## 7.1 Cache Key

Use:

```text
SHA256(source archive bytes)
```

The cache key must be content-based.

The archive registry may separately map:

```text
canonicalArchivePath + size + mtime + file identity
    → lastKnownSHA256
```

This registry is an optimization, not the ultimate authority.

For SNAPSHOT dependencies, correctness must take priority over fingerprinting speed.

## 7.2 Cache Population Algorithm

Pseudocode:

```text
ensureCached(archive):
    fingerprint = identifyArchive(archive)

    acquire per-archive or per-content lock

    if matching valid cache object exists:
        return object

    sha256 = hashArchive(archive)

    if valid object for sha256 exists:
        update registry
        return object

    staging = createUniqueTempDirectory()

    try:
        safelyExtractZip(archive, staging/source)
        validateExtractedTree(staging/source)
        writeCacheManifest(staging)
        publishCacheObject(staging, sha256)
        updateRegistry(archive, sha256)
        return cachedObject
    finally:
        cleanupUnusedStaging()
        release lock
```

## 7.3 ZIP Extraction Security

Never blindly extract ZIP entries by concatenating paths.

Reject:

- Absolute paths.
- `..` traversal.
- Paths resolving outside the extraction directory.
- Symbolic link entries.
- Special device entries.
- Duplicate or ambiguous normalized paths.
- Unsupported entry types.
- Malformed path encodings.
- File/directory collisions.
- Excessive decompression ratios.
- Archives exceeding configured file count or total expanded size.

Use safe path normalization and verify every destination lies underneath the cache object's source root.

The extracted tree must contain ordinary files and directories only.

## 7.4 Cache Immutability

Published cache objects must be immutable by convention.

The plugin must never modify a published cache object's source files.

Cache updates create new content-addressed objects.

A change to a SNAPSHOT artifact produces a new object.

## 7.5 Concurrent Projects

Multiple IDEA processes may request the same archive simultaneously.

Use cross-process file locking or an equivalent atomic publish strategy.

Required properties:

- No duplicate final object.
- No partially published object.
- Concurrent readers only access complete objects.
- Crashed writers leave removable staging directories.
- Different archive extractions can proceed concurrently.

## 7.6 Garbage Collection

Cache GC must not delete arbitrary objects based solely on the current process's open projects.

Project mirrors may survive IDEA shutdown.

The implementation may use a conservative retention policy based on object age and disk quota.

APFS clones remain valid after deletion of the original cache file, but the cache may be needed again for future mirror regeneration.

Therefore GC should default to non-destructive behavior with explicit cleanup or bounded retention.

---

# 8. APFS Materialization

## 8.1 Strategy

Use macOS copy-on-write cloning for individual source files.

The project mirror must contain real regular files.

No directory symlinks are permitted.

## 8.2 Clone Primitive

Preferred native API:

```c
clonefile(sourcePath, destinationPath, 0)
```

A Java/Kotlin implementation may use a native helper, a carefully controlled OS subprocess, or an existing project-native abstraction.

For a CLI-based implementation, macOS provides:

```bash
cp -c source.java destination.java
```

Do not assume the whole directory tree can be cloned as a single metadata operation. Recursion still involves per-entry filesystem operations.

## 8.3 Clone Preconditions

APFS cloning generally requires source and destination to reside on the same compatible filesystem.

The implementation must support:

```text
APFS clone supported → clone
APFS clone unsupported → ordinary copy
```

Fallback must not silently omit files.

The reported materialization mode should be:

```text
APFS_CLONE
REGULAR_COPY
MIXED
```

## 8.4 Avoid Hard Links

Hard links must not be used.

A hard link shares the same underlying file inode, so modifying the project mirror could mutate the global cache content.

APFS clones provide independent file identities with copy-on-write data sharing.

## 8.5 Immutable Reference Content

The mirror is intended to be read-only reference material.

The implementation may remove write permissions from generated files where appropriate, but filesystem permissions are not a complete security boundary.

The primary enforcement is:

- Never edit mirror files programmatically.
- Never sync changes from mirror to cache.
- Replace mirror content only from verified cache objects.
- Document the mirror's generated/read-only nature.

---

# 9. Incremental Synchronization

## 9.1 Planning

Given:

```text
oldManifest
newDependencySnapshot
```

compute:

```text
UNCHANGED
ADDED
REMOVED
UPDATED
FAILED
```

A library is unchanged when its source content identity and intended mirror mapping are unchanged.

GAV alone is insufficient.

## 9.2 Synchronization Algorithm

```text
sync(project):

    snapshot = captureDependencySnapshot()

    desiredEntries = resolveSources(snapshot)

    previousManifest = loadManifest()

    plan = computeSyncPlan(
        previousManifest,
        desiredEntries
    )

    for each added/updated source:
        cacheObject = ensureCached(sourceArchive)

    prepare new mirror generation

    reuse unchanged published files where safe

    clone added/updated files from cache

    validate new generation

    publish generation

    update manifest

    cleanup old generation when safe
```

## 9.3 Publication Semantics

The service should avoid exposing partially materialized files.

A complete generation-based directory replacement is acceptable for the first implementation.

For larger projects, implement a more efficient incremental publication mechanism without sacrificing crash safety.

Potential design:

```text
dependency-sources/          current published generation
dependency-sources.__next/   staging generation
dependency-sources.__old/    previous generation
```

Publication must be performed with carefully ordered filesystem renames.

**Important:** Replacing a non-empty directory tree is not a universally atomic single operation. Do not describe a sequence of renames as globally atomic.

Use a recoverable publish protocol with a journal/state file.

Codex may temporarily hold open old source files during publication. The implementation must tolerate these concurrent readers.

## 9.4 Recovery

On startup:

- Detect incomplete staging generations.
- Validate the currently published manifest.
- Recover from interrupted publication where possible.
- Remove abandoned temporary directories only when known to be plugin-owned.
- Never delete unrelated user files.

## 9.5 Detecting User Files

The mirror directory is plugin-managed.

Before taking ownership of an existing directory:

1. Check for a valid ownership marker.
2. Check the manifest.
3. Refuse destructive cleanup if ownership cannot be proven.
4. Report a conflict and request manual resolution through diagnostics.

Never recursively delete an arbitrary `dependency-sources/` directory merely because it has the expected name.

---

# 10. Lifecycle Integration

## 10.1 Initial Project Open

On project open:

1. Initialize the mirror service.
2. Register project model listeners.
3. Wait until the required project structure is available.
4. Capture a dependency snapshot.
5. Synchronize asynchronously.

Indexing completion should not be treated as an unconditional prerequisite for reading Project Model roots.

The implementation should use appropriate IntelliJ lifecycle and smart-mode APIs only where necessary.

## 10.2 Dependency Changes

Subscribe to Project Model root changes using:

```java
ProjectTopics.PROJECT_ROOTS
```

The event indicates that the model changed; it does not provide a complete dependency diff.

After a suitable debounce interval, capture a new snapshot and compare fingerprints.

## 10.3 Debouncing

Suggested default:

```text
debounce: 1500 ms
```

This is configurable.

Repeated root changes should coalesce into one logical synchronization.

## 10.4 Self-Triggered Events

Excluding the mirror directory may itself trigger a project-root change event.

This can create a feedback loop.

Required protections:

- Track root-model generations.
- Compare dependency snapshots before scheduling extraction.
- Treat mirror exclusion changes as idempotent.
- Avoid resync when only plugin-controlled exclusions changed.

## 10.5 Cancellation

On project close or plugin disposal:

- Stop accepting new project synchronization tasks.
- Cancel queued work.
- Allow safe checkpoints during active extraction/materialization.
- Release locks and resources.
- Preserve the last valid published mirror.

Cancellation must not leave the project mirror in an unrecoverable state.

---

# 11. IDE Exclusion

## 11.1 Requirement

The source mirror must not be indexed as a new project source tree.

IDEA already has access to the original library source roots.

Indexing cloned copies would duplicate work and waste resources.

## 11.2 Implementation

Use the supported IntelliJ Project Model APIs for the target SDK version.

Possible concepts:

```text
ContentEntry
ExcludeFolder
ModifiableRootModel
Write Action
```

For each project content root containing the mirror, add or preserve the necessary excluded root.

Do not overwrite user-defined exclusions.

If the mirror resides outside all module content roots, verify its indexing behavior rather than adding an invalid exclude root.

## 11.3 Critical Invariant

The mirror must be:

```text
visible to macOS filesystem APIs
visible to ripgrep
not indexed as a project source tree by IDEA
not included in Maven compilation
```

The plugin must verify exclusion state after synchronization.

Failure to establish exclusion should generate a warning and may block large materialization according to configuration.

---

# 12. ripgrep and Git Visibility

## 12.1 The Problem

The mirror must be ignored by Git but included by ripgrep.

These are distinct requirements.

## 12.2 Git Rules

The project root `.gitignore` should contain:

```gitignore
/dependency-sources/
```

Existing user content must be preserved.

Do not overwrite the file.

## 12.3 ripgrep Rules

The project root `.rgignore` should contain:

```gitignore
!/dependency-sources/
```

The purpose is to override the Git ignore rule for ripgrep.

The implementation must validate this behavior end-to-end using the installed ripgrep executable.

## 12.4 Validation Commands

```bash
rg --files | rg 'PaymentService.java'
```

```bash
rg -n 'class PaymentService'
```

```bash
git check-ignore dependency-sources/com/company/core/1.2.0/com/company/core/PaymentService.java
```

Expected:

- The first command finds the mirrored Java source.
- The second command searches its contents.
- Git confirms the file is ignored.

## 12.5 Existing Ignore Files

The plugin must handle:

- Existing `.gitignore` content.
- Existing `.rgignore` content.
- Parent-directory ignore files.
- Explicit user exclusions.
- Conflicting patterns.

The plugin should use a bounded managed section when modifying ignore files:

```gitignore
# BEGIN IntelliJ Dependency Source Mirror
!/dependency-sources/
# END IntelliJ Dependency Source Mirror
```

If the user already has an equivalent rule, do not duplicate it.

If an explicit user rule conflicts, report it rather than repeatedly rewriting the user's preferences.

## 12.6 Filesystem Search versus Codex Search

The acceptance criterion is successful native ripgrep traversal and source reading.

This does not guarantee that every proprietary or internal Codex file-discovery mechanism uses identical traversal and ignore semantics.

Codex behavior must therefore be validated empirically using the installed Codex version.

Do not claim compatibility based solely on the presence of `.rgignore`.

---

# 13. CLI Integration

## 13.1 Preserve Existing CLI

Do not break or replace existing PSI commands.

Introduce a small administrative command group.

Suggested command names:

```bash
bridge mirror status
bridge mirror sync
bridge mirror verify
bridge mirror config
bridge mirror cache status
bridge mirror cache gc
```

These commands are primarily for diagnostics and maintenance, not for Codex's ordinary source analysis.

## 13.2 Automatic Mode

Default behavior:

```text
enabled: true
autoSync: true
```

The plugin is responsible for synchronization.

Codex should not have to run:

```bash
bridge mirror sync
```

before using `rg`.

## 13.3 Status

Example:

```text
Dependency Source Mirror

Project: payment-platform
State: READY

Libraries discovered: 248
Libraries with sources: 231
Libraries without sources: 17

Mirrored artifacts: 231
Mirrored Java files: 82,451

Cache hits: 216
Cache misses: 15

Materialization: APFS_CLONE
Last successful sync: 2026-10-24 12:30:00
```

## 13.4 Verify

`bridge mirror verify` must validate:

- Project mirror ownership.
- Manifest integrity.
- Required source paths exist.
- Content hashes where appropriate.
- Git ignore behavior.
- ripgrep visibility.
- IDEA exclusion state.
- Missing or stale dependencies.

## 13.5 JSON Output

Support:

```bash
bridge mirror status --json
```

for scripted diagnostics.

All CLI errors should have stable machine-readable codes.

---

# 14. Configuration

Recommended initial settings:

```yaml
dependencySourceMirror:
  enabled: true
  autoSync: true

  projectDirectory: dependency-sources

  cache:
    directory: "~/Library/Caches/IntelliJDependencySourceMirror"
    contentAddressed: true

  dependencySelection:
    production: true
    test: true
    runtime: true
    includeJdk: false

    include:
      - "*:*"

    exclude: []

  materialization:
    preferred: APFS_CLONE
    fallback: REGULAR_COPY
    includeExtensions:
      - ".java"

  synchronization:
    debounceMillis: 1500
    maxConcurrentExtractions: 4
    maxConcurrentProjects: 2
    autoSyncOnProjectOpen: true
    autoSyncOnRootsChanged: true

  indexing:
    excludeMirrorFromIdea: true

  search:
    manageGitignore: true
    manageRgignore: true
    verifyRipgrep: true

  diagnostics:
    logLevel: INFO
```

Configuration precedence:

```text
Project override
    > IDE global settings
    > Built-in defaults
```

The initial implementation may store settings in IntelliJ PersistentStateComponent instead of YAML.

The YAML above defines logical behavior and does not mandate a particular on-disk settings format.

---

# 15. Performance

## 15.1 Main Principle

Do not treat every IDE Project Model event as a reason to re-extract dependencies.

Most syncs should complete through snapshot comparison and cache reuse.

## 15.2 Target Behavior

For an unchanged project:

```text
No source re-extraction
No full mirror recreation
No dependency re-download
No unnecessary file rewriting
```

For a single dependency version change:

```text
Only affected library entries are materialized or removed
Unrelated mirror files remain unchanged
```

## 15.3 Proposed Performance Targets

These are acceptance targets, not pre-established benchmarks.

On a representative project with approximately 200 source artifacts:

- Unchanged sync: target under 2 seconds after project snapshot acquisition.
- One changed artifact: target under 5 seconds for a typical small or medium source JAR.
- IDE UI blocking: no long-running archive or file-copy operations on the EDT.
- Second project sharing the same dependencies: very high cache reuse.
- Global cache extraction: once per unique source archive hash.

Benchmark against real internal projects and adjust thresholds based on hardware, archive sizes, and filesystem characteristics.

## 15.4 Resource Limits

Configurable limits should cover:

- Maximum archive size.
- Maximum extracted size.
- Maximum ZIP entry count.
- Maximum parallel extractions.
- Maximum mirror disk usage.
- Cache retention.
- Cancellation latency.

Do not allow a malformed archive to exhaust disk space or memory.

---

# 16. Error Handling

## 16.1 Missing Sources JAR

Behavior:

- Record `MISSING_SOURCES`.
- Continue synchronizing other artifacts.
- Preserve the last valid published mirror where appropriate.
- Provide a diagnostic identifying the affected library.

Do not fail the entire project sync.

## 16.2 Archive Changed During Extraction

Detect archive identity changes between snapshot and extraction.

If an archive is replaced or modified:

1. Discard the incomplete extraction.
2. Recapture the relevant source identity.
3. Retry with bounded backoff.

## 16.3 Disk Full

Behavior:

- Abort the affected staging operation.
- Preserve the prior valid mirror.
- Release locks.
- Report a clear error.
- Do not delete user files to make space.

## 16.4 APFS Clone Failure

If clone is unsupported:

- Fall back to regular copy if configured.
- Report materialization mode.
- Preserve source correctness.

If ordinary copy also fails:

- Do not publish incomplete output.

## 16.5 Plugin Crash

The next plugin startup must detect incomplete staging data and recover safely.

## 16.6 Corrupt Cache Object

Never return a cache entry whose manifest or required files fail validation.

Mark it invalid and rebuild from the original archive.

## 16.7 Project Directory Conflict

If `dependency-sources/` already exists and is not identified as plugin-owned:

- Do not overwrite it.
- Report a conflict.
- Require a different configured directory or explicit takeover workflow.

---

# 17. Security

## 17.1 Trust Boundaries

Maven source archives are external input, even when stored in the local Maven repository.

The system must not assume they are safe merely because IDEA attached them.

## 17.2 Required Protections

- ZIP Slip protection.
- Path traversal prevention.
- Symlink and special-file rejection.
- Archive decompression limits.
- Cache ownership checks.
- Controlled file permissions.
- Cross-process lock correctness.
- No shell interpolation of untrusted paths.
- No arbitrary recursive deletion.
- No implicit remote downloads.
- No execution of source archive contents.

## 17.3 Project Root Validation

The project mirror path must resolve to an authorized location.

Prevent configuration from accidentally targeting:

```text
/
~
~/.m2/repository
```

or another sensitive directory.

Project mirror cleanup may only delete paths owned and recorded by the mirror service.

## 17.4 Multi-Process Isolation

A project must not acquire the ability to corrupt another project's mirror through the shared cache.

Cache objects should remain immutable, with local mirrors generated independently.

---

# 18. Testing Strategy

## 18.1 Unit Tests

### Dependency snapshot

- Single-module project.
- Multi-module project.
- Shared library.
- Different versions of the same artifact.
- Missing source root.
- Multiple source roots.
- Non-Maven library.
- Project SDK exclusion.
- Module dependency traversal.

### Cache manager

- Cache hit.
- Cache miss.
- Identical archives at different paths.
- Different archive contents using the same GAV.
- Changed SNAPSHOT.
- Corrupt object.
- Concurrent extraction.
- Interrupted extraction.

### ZIP safety

- `../` entry.
- Absolute entry.
- Symlink entry.
- Duplicate normalized filename.
- Excessive file count.
- Excessive expansion size.
- Malformed archive.

### Mirror planner

- No changes.
- Added dependency.
- Removed dependency.
- Version upgrade.
- Version downgrade.
- Source archive replaced.
- Multiple modules using different versions.
- Missing sources after prior availability.

## 18.2 IntelliJ Integration Tests

Use the project's configured IntelliJ Platform test framework.

Verify:

1. Correct library source-root enumeration.
2. Correct per-module dependency context.
3. Root change notifications trigger synchronization.
4. Plugin-created exclusions do not cause infinite resync.
5. Mirror content remains excluded from IDEA indexing.
6. Project closing cancels background work safely.
7. Existing CLI/PSI functionality remains operational.

The exact test APIs must match the IDE versions supported by the existing plugin.

## 18.3 macOS Filesystem Tests

Required:

- APFS clone behavior.
- Fallback on unsupported filesystem.
- Unicode paths.
- Paths containing spaces.
- Case-insensitive filesystem collisions.
- Long paths.
- File permission failures.
- Cache deletion after mirror creation.
- Concurrent filesystem reads.
- Interrupted directory publication.

## 18.4 ripgrep Integration Tests

Prepare a real temporary Git repository.

Create:

```text
.gitignore
.rgignore
dependency-sources/example/PaymentService.java
```

Assertions:

```bash
rg --files
```

must include `PaymentService.java`.

```bash
rg -n 'class PaymentService'
```

must return the expected source path.

```bash
git status --short --untracked-files=all
```

must not report mirrored Java files.

```bash
git check-ignore dependency-sources/example/PaymentService.java
```

must confirm Git exclusion.

Run these tests with a recent supported ripgrep version on macOS.

## 18.5 End-to-End Test

Construct two independent Maven projects:

```text
Project A → com.example:core:1.0.0
Project B → com.example:core:2.0.0
```

Each dependency has a sources JAR containing an identifiable version-specific implementation.

Test sequence:

1. Open Project A in IDEA.
2. Wait for mirror readiness.
3. Run native `rg` in Project A.
4. Assert only the intended version is present.
5. Open Project B.
6. Verify its different version is mirrored.
7. Verify global cache reuse for shared artifacts.
8. Upgrade Project A's dependency.
9. Reload Maven.
10. Verify the mirror updates automatically.
11. Confirm native `rg` sees the updated version.
12. Restart IDEA.
13. Confirm mirror remains valid and synchronizes.
14. Confirm IDE indexing does not duplicate generated source files.

---

# 19. Observability

Provide structured logs for:

```text
SNAPSHOT_CAPTURED
SYNC_STARTED
SYNC_SKIPPED_UNCHANGED
CACHE_HIT
CACHE_MISS
ARCHIVE_EXTRACTED
MIRROR_CLONED
MIRROR_COPIED
MIRROR_PUBLISHED
SYNC_COMPLETED
SYNC_FAILED
```

Recommended metrics:

```text
snapshotDurationMs
archiveHashDurationMs
extractionDurationMs
materializationDurationMs
publishDurationMs
cacheHitCount
cacheMissCount
mirroredFileCount
mirroredByteCount
missingSourceCount
cloneFallbackCount
```

Logs should avoid printing entire source-file contents.

A CLI diagnostic command should summarize the last completed synchronization and failures.

---

# 20. Rollout Plan

## Phase 1 — Dependency Snapshot

Deliver:

- Project Model enumeration.
- Library source-root mapping.
- Module membership.
- JSON diagnostics.
- Tests for multi-module and multiple versions.

Acceptance:

The plugin can accurately report which attached source archives belong to the current project without running Maven.

## Phase 2 — Global Source Cache

Deliver:

- SHA-256 object store.
- Safe JAR extraction.
- Cache registry.
- Cross-process locking.
- Cache validation.

Acceptance:

The same source archive is extracted once and reused across projects.

## Phase 3 — Project Mirror

Deliver:

- Project filesystem layout.
- APFS Clone.
- Regular copy fallback.
- Manifest.
- Incremental sync.
- Crash recovery.

Acceptance:

Dependency sources are available as regular Java files under the project root.

## Phase 4 — Search and IDEA Integration

Deliver:

- `.gitignore` management.
- `.rgignore` management.
- Native ripgrep verification.
- IDEA Exclude Root management.
- Automated lifecycle integration.

Acceptance:

Default `rg` searches dependency sources, Git ignores them, and IDEA does not duplicate indexing.

## Phase 5 — CLI Diagnostics and Hardening

Deliver:

- Status.
- Sync.
- Verify.
- Cache diagnostics.
- Corruption recovery.
- Performance measurements.
- Failure-injection tests.

Acceptance:

The feature operates reliably across multiple independent projects and IDE restarts.

---

# 21. Definition of Done

The feature is complete only if all the following are true.

- [ ] Uses IDEA Project Model rather than invoking Maven to reconstruct dependencies.
- [ ] Correctly identifies attached source roots.
- [ ] Supports multiple independent projects.
- [ ] Supports different dependency versions across projects and modules.
- [ ] Correctly handles changing SNAPSHOT content.
- [ ] Reuses a global content-addressed source cache.
- [ ] Produces real, ordinary `.java` files.
- [ ] Uses APFS Clone where supported.
- [ ] Falls back safely to regular copy.
- [ ] Does not create directory symlinks in the mirror.
- [ ] Native `rg` can find mirrored source files by default.
- [ ] Native file reads work without a bridge command.
- [ ] Git ignores mirror content.
- [ ] IDEA does not index the generated mirror as project source.
- [ ] Synchronizes after dependency updates.
- [ ] Unchanged syncs avoid unnecessary extraction and copying.
- [ ] Supports cancellation and crash recovery.
- [ ] Protects against malicious ZIP entries.
- [ ] Preserves unrelated user files.
- [ ] Maintains existing Bridge Plugin and CLI functionality.
- [ ] Provides status and verification commands.
- [ ] Includes automated unit, integration, filesystem and end-to-end tests.

---

# 22. Instructions to the Implementing Codex

Implement this design in the existing IntelliJ Bridge Plugin repository.

Before editing code:

1. Inspect the existing plugin's architecture, build system, supported IntelliJ Platform versions, language choices, application/project services, lifecycle infrastructure and CLI command registration.
2. Reuse existing project abstractions and test infrastructure where reasonable.
3. Check actual IntelliJ SDK API availability for the supported versions. Do not assume sample APIs or signatures are compatible.
4. Identify a minimal set of architectural changes that preserve backward compatibility.
5. Create a concrete implementation plan organized by the phases in this document.

During implementation:

- Keep dependency discovery separate from filesystem materialization.
- Keep global cache management separate from project-specific state.
- Never run long extraction/copy operations inside IDEA read or write actions.
- Use IntelliJ concurrency APIs correctly.
- Never perform blocking filesystem operations on the EDT.
- Prefer stable public IntelliJ APIs over internal implementation APIs.
- Ensure all destructive filesystem operations are scoped to plugin-owned paths.
- Ensure generated mirror content is never treated as user-editable application source.
- Implement operational diagnostics and failure recovery alongside the feature.
- Do not silently weaken correctness to improve apparent performance.

After implementation:

1. Run the existing plugin test suite.
2. Add and run the new tests defined above.
3. Perform macOS APFS end-to-end validation.
4. Verify ordinary ripgrep behavior from the actual project root.
5. Verify dependency-version updates after IDEA Maven Reload.
6. Verify IDEA does not duplicate project indexing.
7. Document any deviations from this design and the reasons.
8. Report remaining known limitations.

**Primary success criterion:**

A Codex process working in the Java project can discover and read dependency implementations using its ordinary filesystem tools, with no prior Skill discovery, no explicit Bridge CLI invocation, no source JAR extraction commands and no special ripgrep flags.

**End of Design Document.**