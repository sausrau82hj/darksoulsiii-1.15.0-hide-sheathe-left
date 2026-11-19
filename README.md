# HideStowedHandItem — Dark Souls III 1.15.0

While you two-hand one hand's weapon, the **other hand's sheathed model is no longer drawn on your back**. The behaviour is symmetric: two-hand the right weapon and the left item disappears; two-hand the left weapon and the right item disappears. Works for anything hand-equippable: weapons, shields, catalysts, bows, staves.

> Dark Souls III **1.15.0** (Steam, 64-bit). Local memory + model changes only. Online play has a server-side save sanity check — use it offline.

## What it does

| Situation | Vanilla | With this mod |
|---|---|---|
| Right-hand weapon **two-handed**, left hand occupied | left item hangs on your back | left item is **not drawn** |
| **Left-hand** weapon two-handed, right hand occupied | right item hangs on your back | right item is **not drawn** |
| The stowed item **in use** (blocking / swinging) | drawn normally | **drawn normally** (unaffected) |
| **Paired weapon** held one-handed (either hand) | spare model sits on the body | spare model **not drawn** (`PairedFields` can hide only the blade and keep the scabbard) |

**How:** the `leftHang_0..3` / `rightHang_0..3` and `right_1..3` / `left_1..3` fields of `WepAbsorpPosParam` are pointed at a **parking dummy** (id `382`, with `383` as a spare) that sits **1000 m below the character**. The model is still attached, just far outside the world — so nothing is drawn, and because the dummy is a *real* dummy the engine builds a valid matrix (an invalid poly like `-1` draws black triangles and full-screen flicker).

The parking dummies are appended to **every character model** (`chr\*.chrbnd.dcx`), so no character — player or NPC — can ever resolve a missing dummy.

## Requirements

* Dark Souls III 1.15.0 (Steam, 64-bit).
* A **loose-file setup** (`Game\chr\*.chrbnd.dcx` exists — the normal state after unpacking with UXM, which model mods require). `tools\ds3tae.exe` edits those files in place.
* Windows. The patcher is a self-contained binary; .NET is only needed if you rebuild it.

## Install

No installers in this repository — four manual steps. The game must be closed.

1. **Back up** `Game\chr\*.chrbnd.dcx` (about 340 MB for the full 165-model set) and, if it already exists, `Game\xinput1_3.dll`.
2. **Copy the payload** into `Game\`: `HideSheathedLeft.dll`, `HideSheathedLeft.ini`, `xinput1_3.dll`.
3. **Give every character model the parking dummies** (from the folder you unpacked this repository into, with `<game>` = `...\DARK SOULS III\Game`):

```
tools\ds3tae.exe parkdir "<game>\chr" INPLACE "382,383" "0,-1000,0" 0 "chrbnd"
```

4. **Prove it took** — the two numbers must match:

```
tools\ds3tae.exe verify "<game>\chr" chrbnd "382,383"
```

`scanned N file(s), N with an FLVER … files still carrying 382,383: N` means every model is patched. A smaller second number means some model has no dummy, which is exactly what produces the black triangles; re-run step 3 (already-patched files are skipped, so it is harmless to repeat). Add `DRYRUN` as the last argument of step 3 to see what it would do without writing.

`xinput1_3.dll` only exists to make the game load `HideSheathedLeft.dll`: it forwards the XInput exports to the system DLL, so controllers are unaffected. Only one `xinput1_3.dll` can live in the game folder — if another mod already owns it, rename that file first, or load `HideSheathedLeft.dll` with ModEngine and skip this file.

Step-by-step with troubleshooting: [`docs/MANUAL-INSTALL.txt`](docs/MANUAL-INSTALL.txt).

## Configuration

`HideSheathedLeft.ini` is re-read every second: edit, save, done — no restart. A missing file means all defaults.

| Key | Default | Meaning |
|---|---|---|
| `HideLeftHang` | `1` | hide the sheathed left-hand item (reached by two-handing the **right** weapon) |
| `HideRightHang` | `1` | mirror image: hide the sheathed right-hand item (reached by two-handing the **left** weapon) |
| `HideBothPaired` | `1` | hide the spare model of a one-handed paired weapon, **both hands** |
| `PairedFields` | `1,2,3` | which models of a paired weapon to hide: `1` = blade only (keeps the scabbard), `none` = leave paired weapons alone |
| `HidePoly` | `382` | parking dummy id (`383` spare) |
| `IntervalSeconds` | `1` | how often the parameter values are re-asserted |
| `Rescan` | `1` | `0` = stop the periodic address-space re-scan (see Performance) |
| `ScanMapped` | `0` | `1` = also consider MEM_MAPPED regions (diagnostic) |
| `Probe` | `0` | `1` = observation only, writes nothing |
| `Log` | `0` | `1` = write `HideSheathedLeft.log` next to the ini (diagnostics only; off by default) |
| `LoadCore` | `1` | `0` = the loader shell loads nothing (lets you keep `xinput1_3.dll` in place while the mod is off) |
| `ExtraMods` | *(empty)* | must stay empty — loading additional DLLs on 1.15.0 crashes the game |

`HideLeftHang=0` restores vanilla for the right-hand two-hand stance only; `HideRightHang=0` does the same for the left-hand stance. The older key `HidePairedOffhand` is still read when `HideBothPaired` is absent.

## Uninstall

1. Delete `Game\HideSheathedLeft.dll` and `Game\HideSheathedLeft.ini`.
2. Put your own `Game\xinput1_3.dll` back (or delete the file if you had none).
3. Put the character models back from your backup. Without a backup, strip the dummies instead:

```
tools\ds3tae.exe stripdir "<game>\chr" INPLACE "382,383" "chrbnd"
```

A stripped bundle is structurally identical but **not byte-identical** (the DCX is re-compressed). For exact bytes use your backup, or delete the loose `chr\*.chrbnd.dcx` files and let the game read the packed ones. Prove the result with the `verify` command from the install section — a clean game ends in `verify: CLEAN`.

To only switch the effect off, set `HideLeftHang=0` and `HideRightHang=0` in the ini; that applies within a second and leaves the install in place.

## Performance

The parameter writes are the cheap part: they only happen when a value actually differs, so steady state is a handful of memory reads per second. The expensive part is finding the parameter table — the mod walks the process's memory regions and copies their **resident** pages looking for the table name. That runs once at start-up, and again only when the addresses it holds go stale (`Rescan=1`, the default, keeps the effect self-healing).

* An earlier build read **non-resident** pages too, which faults paged-out data back into RAM: the game's working set jumped by ~4 GB (6007 MB vs 1634 MB vanilla) with heavy stutter. This build is resident-only.
* If you see stutter, set **`Rescan=0`**. No address-space pass runs after the start-up scan and the mod is reduced to those few reads per second. Trade-off: if the engine rebuilds the table mid-session, the effect can stay off until the next launch.
* Nothing is written to disk: the mod keeps no log file, so a stutter that survives `Rescan=0` is not the parameter scan.

## Known limitations

* **NPCs hide the item too.** The parameter table is shared between the player and NPCs, and the parking dummy is valid for every character model, so an NPC in the same stance simply does not draw its sheathed item instead of producing artefacts. Player-only behaviour would require patching the engine's attach resolver.
* **Rare "silent no-op".** The engine keeps several copies of the parameter table in memory and the mod can only rewrite the ones it finds, so occasionally the copy the engine reads is missed: the item stays on your back (no artefacts) until the next launch, because the engine caches resolved attach values per character.
* **Black triangles / flicker** mean a character model has no dummy `382`/`383` and the engine built a degenerate attach matrix — run the `verify` command and re-run the patch if the numbers disagree.
* After another mod replaces files in `chr\`, run the `parkdir` command again; already-patched files are skipped.

## Repository layout

```
HideSheathedLeft.dll / .ini   mod core and its configuration
xinput1_3.dll                 loader shell (forwards the system XInput)
tools\ds3tae.exe              patch / verify / strip tool (+ SoulsFormats.dll)
docs\MANUAL-INSTALL.txt       step-by-step manual install
```

## Notes

* This repository ships **no game assets**: `ds3tae.exe` only appends two dummies to **your own** `chr\*.chrbnd.dcx` files and verifies that everything else stays unchanged.
* The table scan only reads pages the OS currently holds resident, so it cannot pull the game's paged-out resource pools back into RAM.
* Backups are your uninstall path — 340 MB for the full `chr\` set, and the reason step 1 exists.

## License

MIT — see [`LICENSE`](LICENSE). Not affiliated with FromSoftware or Bandai Namco; single-player, offline use only.
