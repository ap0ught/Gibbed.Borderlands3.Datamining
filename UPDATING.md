# Updating the EverSpace 2 SDK

## Background

The SDK in this repository was generated in **August 2020** from the **EverSpace 2 Prototype**,
which ran on **Unreal Engine 4.22**. The tool used was the
[satisfactorymodding/UnrealProjectGenerator](https://github.com/satisfactorymodding/UnrealProjectGenerator)
as documented in RussellJerome's original video.

As of **April 29, 2024**, EverSpace 2 migrated to **Unreal Engine 5.3** with the free
"Incursions" update. Every offset, class layout, and vtable entry in the current SDK is
therefore stale and must be regenerated.

---

## What Changed

| Property | Original SDK (2020) | Current Game (2024+) |
|---|---|---|
| Game build | Prototype | Full release + Incursions DLC |
| Unreal Engine | UE 4.22 | UE 5.3 |
| SDK generator used | satisfactorymodding UE4 tool | Dumper-7 (UE4 + UE5) |
| PDB available | Yes (prototype) | No (shipping build) |
| GNames layout | Classic `GNames` array | `FNamePool` (UE5) |

---

## Requirements

- A legally owned copy of **EverSpace 2** (current version, post-Incursions)
- [Dumper-7 by Encryqed](https://github.com/Encryqed/Dumper-7) — supports UE4 and UE5
- A DLL injector (e.g. [Process Hacker](https://processhacker.sourceforge.io/))
- **Visual Studio 2022** (to build Dumper-7)
- Basic knowledge of DLL injection

---

## Step 1 — Build Dumper-7

```
git clone https://github.com/Encryqed/Dumper-7.git
```

1. Open `Dumper-7.sln` in Visual Studio 2022
2. Set configuration to **Release / x64**
3. Build — produces `Dumper-7.dll`

Dumper-7 auto-detects the UE version and whether the game uses `GObjects`/`GNames` or
the newer `FNamePool` (required for UE5), so no manual offset hunting is needed in most cases.

---

## Step 2 — Launch EverSpace 2

Start `EverSpace2.exe` and wait until you reach the **main menu or are in-game**.
The game process must be running before injection.

---

## Step 3 — Inject the DLL

1. Open your DLL injector
2. Attach to the process: `EverSpace2-Win64-Shipping.exe`
3. Inject `Dumper-7.dll`
4. A console window appears showing dump progress
5. When complete the SDK is written to:
   ```
   %localappdata%\Dumper-7\EverSpace2-SDK\
   ```

---

## Step 4 — Verify the Output

Confirm that the output folder contains core modules, e.g.:

```
SDK.hpp
SDK/ES2_Basic.hpp
SDK/ES2_CoreUObject_classes.hpp
SDK/ES2_Engine_classes.hpp
SDK/ES2_AIModule_classes.hpp
SDK/ES2_<game-specific>_classes.hpp
```

---

## Step 5 — Update This Repository

```bash
# Remove the stale SDK
rm -rf SDK/ SDK.hpp

# Copy in the freshly generated files
cp -r "%localappdata%\Dumper-7\EverSpace2-SDK\SDK"     ./SDK
cp    "%localappdata%\Dumper-7\EverSpace2-SDK\SDK.hpp" ./SDK.hpp

git add .
git commit -m "Regenerate SDK for EverSpace 2 UE5.3 (Incursions)"
git push
```

Update this file with the new game version, UE version, and the date of regeneration.

---

## UE5-Specific Notes

- **`FNamePool`** replaces the old `GNames` flat array. Dumper-7 handles this automatically.
  If you see garbled names, refer to the
  [Dumper-7 manual offset section](https://github.com/Encryqed/Dumper-7#finding-offsets).
- **No PDB** in the shipping build. Dumper-7 does not require one — it pattern-scans the
  reflection metadata baked into the binary.
- **Do not mix** old (UE4.22) headers with new (UE5.3) ones. Core types like `FString`,
  `TArray`, and `FName` have layout differences.

---

## Alternative Tools

| Tool | Notes |
|---|---|
| [Dumper-7](https://github.com/Encryqed/Dumper-7) | **Recommended.** UE4 + UE5, auto-detects everything |
| [UE-Dumper by McDaived](https://github.com/McDaived/UE-Dumper) | UE4 4.20 → UE5.3, Windows only |
| [ue4genny by cursey](https://github.com/cursey/ue4genny) | Late UE4 + UE5, requires source access |

---

## References

- RussellJerome's original video — prototype, UE4.22, satisfactorymodding tool
- [Encryqed/Dumper-7](https://github.com/Encryqed/Dumper-7)
- [EVERSPACE 2 UE5 announcement](https://www.unrealengine.com/tech-blog/everspace-2-sets-a-course-for-the-future-through-unreal-engine-5)
- Original upstream SDK: [RussellJerome/EverSpace2-SDK](https://github.com/RussellJerome/EverSpace2-SDK)
