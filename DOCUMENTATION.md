# Gibbed's Borderlands 3 Datamining Tools — Documentation

## Table of Contents

1. [Overview](#overview)
2. [Repository Structure](#repository-structure)
3. [How It Works](#how-it-works)
4. [Projects](#projects)
   - [UnrealScriptFormats](#unrealscriptformats)
   - [Borderlands3ScriptFormats](#borderlands3scriptformats)
   - [Gibbed.Unreflect (submodule)](#gibbed-unreflect-submodule)
   - [Gibbed.IO (submodule)](#gibbed-io-submodule)
   - [NDesk.Options (submodule)](#ndeskoptions-submodule)
5. [Building](#building)
6. [Usage](#usage)
7. [Security Considerations](#security-considerations)
8. [Compatibility: BL3-Only vs. Other Unreal Engine Games](#compatibility-bl3-only-vs-other-unreal-engine-games)
9. [Adapting to Other Games (e.g., Everspace 2)](#adapting-to-other-games-eg-everspace-2)
10. [License](#license)

---

## Overview

This repository contains C# datamining tools written by Rick ("gibbed") to extract and structure
game data from **Borderlands 3** (BL3). The extracted data is used to populate the companion
project [Gibbed's Borderlands 3 Save Editor](https://github.com/gibbed/Gibbed.Borderlands3).

Two data extraction methods are supported:

- **In-process memory dumping** — reads live data from a running BL3 process using
  `Gibbed.Unreflect`.
- **File-based parsing** — deserializes cooked Unreal Engine binary asset files (`.uasset` /
  `.uexp`) directly from disk.

---

## Repository Structure

```
Gibbed.Borderlands3.Datamining/
├── bin/
│   └── dumps/                  # Git submodule — pre-generated data dumps
│                               # (github.com/gibbed/Borderlands3Dumps)
├── projects/
│   ├── Borderlands3ScriptFormats/   # BL3-specific UObject class definitions
│   │   ├── Borderlands3ScriptFormats.csproj
│   │   └── Script/
│   │       ├── GbxGameSystemCore/   # Gearbox core game systems (parts, attributes, UI stats)
│   │       ├── GbxInventory/        # Inventory/loot system (balance data, parts, manufacturers)
│   │       ├── GbxRuntime/          # Gearbox runtime data assets
│   │       └── OnlineSubsystemUtils/ # DLC/downloadable content data
│   ├── Gibbed.IO/               # Git submodule — binary I/O primitives (endianness, streams)
│   ├── NDesk.Options/           # Git submodule — command-line argument parsing
│   ├── UnrealScriptFormats/     # Game-agnostic Unreal Engine 4 binary format library
│   │   ├── UnrealScriptFormats.csproj
│   │   ├── IStubbed.cs          # Marker interface for partially-implemented classes
│   │   ├── Name.cs              # UE4 FName struct (id + number + string lookup)
│   │   ├── ObjectReference.cs   # UE4 object index (import/export table pointer)
│   │   ├── PropertyBag.cs       # Base class for UObjects with tagged properties
│   │   ├── PropertyBagHelper.cs # Tagged property deserialization loop
│   │   ├── PropertyTag.cs       # UE4 FPropertyTag (property name, type, size metadata)
│   │   ├── SerializationHelper.cs # Bool tag extraction helper
│   │   ├── TaggedArray.cs       # UE4 TArray with inner PropertyTag header
│   │   ├── Text.cs              # UE4 FText (localized string)
│   │   ├── Serialization/
│   │   │   ├── IUnrealSerializable.cs  # Interface: objects that know how to serialize themselves
│   │   │   ├── IUnrealSerializer.cs    # Interface: serializer providing typed read methods
│   │   │   └── UnrealSerializationMode.cs # Enum: Loading / Saving
│   │   └── Script/
│   │       ├── CoreUObject/    # UE4 core reflection types (UObject, UStruct, UClass, UFunction…)
│   │       └── Engine/         # UE4 engine asset types (DataAsset, DataTable, BlueprintGeneratedClass)
│   └── Unreflect/              # Git submodule — in-process game reflection/memory dumping
│                               # (github.com/gibbed/Gibbed.Unreflect)
├── .gitignore
├── .gitmodules
├── README.md
└── DOCUMENTATION.md            # This file
```

---

## How It Works

### In-process memory dumping

`Gibbed.Unreflect` injects into (or attaches to) a running BL3 process and uses the game's own
Unreal Engine reflection data (`GUObjectArray`, `GNames`, class hierarchies) to enumerate and
extract live object data. This requires the game to be running and is tightly coupled to a
specific build version of BL3.

### File-based parsing

The `UnrealScriptFormats` library deserializes Unreal Engine 4 cooked binary package files.
The standard UE4 cooked format consists of:

- A **package header** containing import and export tables, name table (`GNames`), and
  dependency information (parsed by `Gibbed.Unreflect` or an upstream tool).
- **Export data** — each export is a serialized UObject. The data is read using the tagged
  property format (see `PropertyBagHelper`, `PropertyTag`).

Classes in `Borderlands3ScriptFormats` mirror BL3's actual UObject subclasses and override
`SerializeProperty` to handle each named property field. Unknown properties are stored raw as
`byte[]` in `UnknownProperties`.

---

## Projects

### UnrealScriptFormats

A **game-agnostic** C# library for Unreal Engine 4 binary data. It implements:

| Type | Purpose |
|------|---------|
| `IUnrealSerializable` | Any type that knows how to read/write itself via `IUnrealSerializer` |
| `IUnrealSerializer` | Serializer interface — provides typed `Serialize(ref T)` methods and a `LookupName` name table |
| `Name` | `FName` — a UE4 string identifier stored as `(id, index)` and resolved against a name table |
| `ObjectReference` | UE4 package object reference (negative = import, positive = export, zero = null) |
| `PropertyTag` | `FPropertyTag` metadata (property name, type, size, GUID, inner types for containers) |
| `PropertyBag` | Base class for UObjects; reads a sequence of tagged properties until `"None"` sentinel |
| `PropertyBagHelper` | Drives the tagged-property deserialization loop; supports a "safe" mode that buffers each property into a `MemoryStream` for isolated parsing |
| `TaggedArray` | `TArray` with a leading count and an inner `FPropertyTag` header |
| `Text` | `FText` (localizable string) |

The `Script/CoreUObject` and `Script/Engine` subdirectories implement foundational Unreal types
(`UObject`, `UStruct`, `UClass`, `UFunction`, `UProperty`, `UScriptStruct`,
`BlueprintGeneratedClass`, `DataAsset`, `DataTable<TRow>`).

> **Note:** Only the **Loading** direction is implemented. Saving/writing raises
> `NotSupportedException` or `NotImplementedException` throughout.

### Borderlands3ScriptFormats

BL3-specific `PropertyBag` subclasses that map to Gearbox's actual UObject hierarchy.
Each class declares its fields and overrides `SerializeProperty` to match property names exactly
as they appear in the cooked assets.

Key namespaces and classes:

| Namespace | Class | Purpose |
|-----------|-------|---------|
| `GbxInventory` | `InventoryBalanceData` | Per-item balance: rarity, base balance chain, part set, manufacturers |
| `GbxInventory` | `InventoryData` | Item type: display name, naming strategy, actor class |
| `GbxInventory` | `InventoryPartSetData` | Weighted part pool for an item |
| `GbxInventory` | `InventoryPartData` | A single inventory part definition |
| `GbxInventory` | `InventoryRarityData` | Rarity tier definition |
| `GbxInventory` | `ManufacturerData` | Manufacturer definition |
| `GbxInventory` | `InventoryNamingStrategyData` | How item names are assembled |
| `GbxGameSystemCore` | `ActorPartSelectionData` | Part-selection logic for spawned actors |
| `GbxGameSystemCore` | `AttributeEffectData` | Stat modifier definitions |
| `GbxGameSystemCore` | `UIStatData` | UI stat display data |
| `GbxRuntime` | `GbxDataAsset` | Gearbox base data asset |
| `OnlineSubsystemUtils` | `DownloadableContentData` | DLC package metadata |

### Gibbed.Unreflect (submodule)

Source: `https://github.com/gibbed/Gibbed.Unreflect`

Provides in-process reflection of a running Unreal Engine game. It reads `GUObjectArray` and
`GNames` from process memory to enumerate live UObjects and walk the class hierarchy. Used for
generating the initial data dumps that seed the `bin/dumps` submodule.

> **Security-sensitive** — see [Security Considerations](#security-considerations).

### Gibbed.IO (submodule)

Source: `https://github.com/gibbed/Gibbed.IO` (branch: `vs2017`)

Low-level binary I/O helpers. Provides:

- Endianness-aware primitive readers/writers (`Endian` enum, `StreamHelpers`)
- Utilities referenced by `IUnrealSerializer` implementations

### NDesk.Options (submodule)

Source: `https://github.com/gibbed/NDesk.Options` (branch: `vs2017`)

A command-line option parser used by the tool executables (not visible in this subgraph but
referenced by the tooling projects).

---

## Building

> A `.sln` file is not present in the repository root — it is either in a private directory or
> not yet committed. To build manually:

1. Ensure submodules are initialized:
   ```sh
   git submodule update --init --recursive
   ```
2. Open the individual `.csproj` files in Visual Studio 2017+ or use `dotnet build` if projects
   have been migrated to SDK-style format.
3. Target framework is likely **.NET Framework 4.x** (inferred from `vs2017` submodule branches
   and file structure). Verify in `.csproj` files after checking out.

---

## Usage

The original README states: *TODO.*

Based on the code, the general flow is:

1. Either run the in-process dumper against a live BL3 process **or** point the tool at
   extracted cooked asset files (`.uasset` / `.uexp`).
2. The tool deserializes the relevant `InventoryBalanceData`, `DataTable`, and related objects.
3. Output is presumably JSON or a similar format consumed by the BL3 Save Editor.

Refer to `bin/dumps` (the `Borderlands3Dumps` submodule) for example pre-generated output.

---

## Security Considerations

The following areas require attention when maintaining or extending this project:

### 1. Process Memory Reading (Anti-Cheat Risk)
`Gibbed.Unreflect` reads from a live game process. Modern anti-cheat systems (Easy Anti-Cheat,
BattlEye) may detect and flag this. **Use only in offline/single-player mode** and be aware
that future BL3 patches or anti-cheat updates may break the tool or trigger account actions.

### 2. Unbounded Memory Allocation from Untrusted Data
In `PropertyBagHelper.SerializeTaggedProperties`, `tag.Size` is read directly from the binary
stream and used to allocate a `byte[]`:
```csharp
var bytes = new byte[tag.Size];
```
A maliciously crafted or corrupted asset file could set `tag.Size` to a very large value,
causing an `OutOfMemoryException` or denial-of-service. **Mitigation**: add a maximum size
sanity check before allocating (e.g., reject sizes > 256 MB).

### 3. No Input Validation on Deserialized Counts
In `DataTable<TRow>.Serialize` and `TaggedArray.Serialize`, the item `count` is read from the
stream without bounds checking before iterating. A corrupt file could cause very long loops.
**Mitigation**: validate `count` against a reasonable maximum.

### 4. Write Path Not Implemented — Deserialization-Only Codebase
All `Saving` code paths throw `NotSupportedException`/`NotImplementedException`. This is
expected for a read-only datamining tool, but any future attempt to add write support should
be done carefully and tested against a safe copy of game files.

### 5. Outdated Submodule Branches
`Gibbed.IO` and `NDesk.Options` are pinned to the `vs2017` branch. These should be reviewed
for upstream security fixes or updated to their latest versions. Run `git submodule update`
periodically and review changelogs.

### 6. No .NET Version Pin
The project targets an older .NET Framework version implied by Visual Studio 2017 references.
Migrating to a current .NET SDK (e.g., .NET 8 LTS) would bring modern runtime security
improvements (bounds checking, span-based I/O, etc.).

### 7. `IStubbed` Marker Interface
Classes implementing `IStubbed` (e.g., `InventoryData`) are partially implemented — some
fields are noted as `// TODO`. Stub classes silently drop unknown data. This is a correctness
concern rather than a security risk, but it means some extracted data may be incomplete.

---

## Compatibility: BL3-Only vs. Other Unreal Engine Games

**Short answer: The core format library is generic UE4, but the schema definitions are
BL3-specific.**

### What is game-agnostic

The `UnrealScriptFormats` library implements the **standard Unreal Engine 4 tagged property
serialization format**. This format is shared across all UE4 (and many UE5) games that use
cooked `.uasset`/`.uexp` files. Specifically, `PropertyTag`, `PropertyBag`, `Name`,
`ObjectReference`, `DataTable`, and `DataAsset` all mirror standard UE4 types.

In principle, `UnrealScriptFormats` could be reused for any UE4 game's cooked assets,
as long as the underlying format version matches.

### What is BL3-specific

Everything in `Borderlands3ScriptFormats` is tightly coupled to Gearbox's proprietary class
hierarchy:

- `GbxInventory.*` — Gearbox's inventory/loot system
- `GbxGameSystemCore.*` — Gearbox's attribute and actor-part system
- `GbxRuntime.GbxDataAsset` — Gearbox's base data asset class
- `OnlineSubsystemUtils.DownloadableContentData` — BL3's DLC system

These class names, property names, and field layouts **do not exist** in other UE4 games. Other
games have their own proprietary class hierarchies and property layouts.

### The `Gibbed.Unreflect` memory dumping path

The in-process dumper uses GNames/GUObjectArray layouts that differ between UE4 versions and
between games compiled with different settings. It would need re-calibration for any new game.

---

## Adapting to Other Games (e.g., Everspace 2)

**Everspace 2** (by ROCKFISH Games) is built on Unreal Engine 4 (approximately UE4.25–UE4.27).

### What would transfer

| Component | Transferable? | Notes |
|-----------|--------------|-------|
| `UnrealScriptFormats` library | ✅ Yes | Standard UE4 format; reusable as-is |
| `Gibbed.IO` | ✅ Yes | Generic binary I/O |
| `PropertyBag` / `PropertyTag` deserialization | ✅ Yes | Same tagged format |
| `DataTable<TRow>`, `DataAsset` | ✅ Yes | Standard UE4 engine types |
| `Borderlands3ScriptFormats` | ❌ No | BL3-specific class definitions |
| `Gibbed.Unreflect` offsets | ❌ No | Needs re-calibration for ES2 build |

### What would need to be created

1. **`Everspace2ScriptFormats` project** — A new project mirroring the structure of
   `Borderlands3ScriptFormats`, with C# classes matching ROCKFISH's actual UObject subclasses
   (ship components, equipment, items, etc.). These property names and layouts would need to be
   reverse-engineered from ES2's cooked asset files.

2. **Name table reader** — The package header parser (upstream of `UnrealScriptFormats`) needs
   to supply the name table (`GNames`) for `Name.LookupName`. This is currently handled by
   `Gibbed.Unreflect` or a separate package loader not visible in this subgraph.

3. **UE version compatibility check** — Confirm ES2's exact UE4 version, as the tagged property
   format has minor variations across UE4 versions (e.g., format version flags, property GUID
   presence).

### Recommended alternative tools for Everspace 2

For **browsing** ES2 game data without custom code, established community tools already handle
arbitrary UE4/UE5 games and may be faster to use:

- [**FModel**](https://fmodel.app/) — UE4/UE5 asset browser with class schema support
- [**UEViewer / umodel**](https://www.gildor.org/en/projects/umodel) — UE asset viewer
- [**CUE4Parse**](https://github.com/FabianFG/CUE4Parse) — C# library for parsing UE4/UE5 packages

If the goal is to build a save editor or data exporter specifically for Everspace 2 (similar to
what this repo does for BL3), this repo's architecture is a reasonable model to follow, but
significant new work on schema definitions would be required.

---

## License

The code is licensed under the **zlib license** (a permissive open-source license). The full
license text appears at the top of every source file:

> Permission is granted to anyone to use this software for any purpose, including commercial
> applications, and to alter it and redistribute it freely, subject to the following
> restrictions:
> 1. The origin of this software must not be misrepresented.
> 2. Altered source versions must be plainly marked as such.
> 3. This notice may not be removed or altered from any source distribution.

Copyright © 2019 Rick (rick 'at' gibbed 'dot' us)
