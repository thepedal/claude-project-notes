# ReBuzz Managed Machine Notes — Renaming (machines & repos)

Source: naming/rename planning during Pedal Juno106 v1.8, following the
trademark findings in `ReBuzz_ManagedMachine_Notes_Licensing.md` (§3). This
file is the **how**: candidate names, the surfaces a rename touches, the
saved-song cost, and the GitHub repo mechanics. The **why** (which machines
carry marks, the disclaimer, the descriptive-use template) lives in the
Licensing notes §3.

GitHub behaviour below was checked 2026-07-11 against GitHub's docs; treat as a
snapshot. `Licensing §N` → `..._Notes_Licensing.md`; `Build §N` →
`..._Notes_Build.md`.

**Nothing here is decided.** The names in §1 are proposals; the rename itself
is an open decision (Licensing §6).

---

## 1. Candidate replacement names

Template (from the two machines that already do it right): an original or
functional name, with the heritage carried in the *description* ("emulates the
X"), plus the Licensing §3.1 disclaimer. Follow `pedal-Faze-R` and
`pedal-invFFT`.

| Current | Mark | Proposed (primary) | Alternates | Rationale |
|---|---|---|---|---|
| pedal-juno106 | Roland Juno-106 | **Pedal Hexa** | Lune, Chorale | Hexa = 6-voice poly; Lune = Juno/moon nod; Chorale = its chorus |
| pedal-sh101 | Roland SH-101 | **Pedal Monoline** | Mono-1, Grip | mono bass/lead voice; "Grip" = SH-101 mod grip |
| pedal-M1 | Korg M1 | **Pedal Rompler** | PCM-Duo, Rom-1 | "rompler" is the generic category the M1 defined |
| pedal-dly-PCM41 | Lexicon PCM 41 | **Pedal Reflex** | Deltatime, DigiDelay | echo/reflection; avoid "PCM"/"Lexi" |
| pedal-zplane | E-mu Z-Plane | **Pedal Morph** | Quadra, Poleshift | morphing filter; "z-plane" is arguably generic DSP, but Morph removes the argument |
| pedal-plaits | Mutable Plaits | **Pedal Weave** | Macro | keeps the braid metaphor; **low priority** — the real action is MIT attribution (Licensing §4), vendor is defunct/open |

Before committing any name, **check availability** (§5): no collision with an
existing Buzz machine, a `thepedal` repo, or a well-known plugin. The proposals
are generic English words by design (low trademark risk), but a direct
collision is worth avoiding.

---

## 2. Two independent renames — don't conflate them

A machine's public identity has two separate names, and they can be changed
independently:

- **The GitHub repo name** (`thepedal/pedal-juno106`). Renaming this is
  **trivial and safe** — GitHub redirects cover it (§4). Cosmetic/hosting only;
  does not touch any user's ReBuzz install.
- **The machine (library) name** — what ReBuzz uses to identify the machine in
  a song. This is the one with a **saved-song cost** (§3). Driven by the
  `AssemblyName` / `MachineDecl.Name`, surfaced as `<Library>` in `.bmxml`.

You can do the repo side now and the machine side later, or together — they
don't depend on each other. The repo rename also removes the trademarked model
number from the public URL, which is part of the point.

---

## 3. Machine rename — surfaces to change, and the song cost

### 3.1 Every surface a machine rename touches

Using Juno → `Pedal Hexa` as the worked example:

1. **`.csproj` `AssemblyName`** — `Pedal Juno106.NET` → `Pedal Hexa.NET`. Must
   still end in `.NET` (Build §2 — the browser display name comes from the DLL
   filename).
2. **`MachineDecl.Name` and `ShortName`** — `Pedal Juno106` → `Pedal Hexa`.
3. **Preset file** — `Pedal Juno106.prs.xml` → `Pedal Hexa.prs.xml` (Build §3.1
   auto-load name = machine name).
4. **`<Preset Machine="…">` inside the .prs.xml** — must equal the new
   `MachineDecl.Name` exactly, or presets silently don't load (Build §3.2). Easy
   to miss.
5. **`.csproj` deploy target** — the `_PresetPath` / bundle name (Build §3.5.1)
   → new `.prs.xml` name.
6. **`gen_presets.py`** — the emitted `Machine="…"` attribute and the output
   filename.
7. **About box** — display-name string, the title, and the repo URL
   (AboutWindow §1.4).
8. **README / SESSION_STATE** — titles and references.
9. **Roster row** — the Repo/name entry (auto-picked-up on a fresh API pull, §6).

**Can stay unchanged:** the C# class name and namespace (`PedalJuno106`) — they
never surface to the user and are independent of `AssemblyName`/`RootNamespace`.
Leaving them avoids a large, pointless source churn.

### 3.2 The saved-song cost

ReBuzz identifies a machine in a `.bmxml` by its **library name** (the
`<Library>` string, e.g. `Pedal Juno106`). Rename the machine and every
existing song referencing the old name loads it as a **missing machine** — the
DSP is fine, the reference is stale.

Mitigations, in order of preference:
- **Peer-controlled songs swap easily.** In `amanita_pantherina_041` the
  synths are driven by control machines (chord/peer), so replacing the old
  instance with the renamed one and re-pointing the controller is quick.
- **Rename going forward only** — leave shipped songs on the old-named machine
  (keep the old DLL installed alongside), name new machines/songs with the new
  name. No break, but two names coexist for a while.
- **Alias/remap** — *unconfirmed*: check whether ReBuzz supports a machine
  name-alias/remap so old `<Library>` references resolve to the new machine. If
  it does, that's the clean path for old songs. (Open item — not yet verified.)

Order the work so the preset `Machine=` attribute (3.4) and the deploy name
(3.5) change together with `MachineDecl.Name`; a mismatch there is the classic
"machine renamed, presets vanished" bug.

---

## 4. GitHub repo rename — mechanics

Renaming a repo is Settings → **Repository name**, one repo at a time (no bulk
button; `gh repo rename` scripts it for several).

What GitHub does on rename:
- **Redirects everything except project-site URLs** to the new name, and keeps
  the redirects **indefinitely**. Web links, and `git clone`/`fetch`/`push`
  against the old URL, keep working as if made on the new location. Release-asset
  URLs redirect too.
- **Exceptions that do NOT redirect:** GitHub **Actions** (a workflow calling an
  action by its old repo path fails with "repository not found") and GitHub
  **Pages** project-site URLs. Our machine repos publish neither, so this is
  almost certainly moot — glance at any repo that has workflows.
- **Don't recycle the old name:** creating a new repo with the old name kills
  the redirect.
- **Update local clones:** `git remote set-url origin <new>`. The `gh` CLI can
  silently return empty results in scripts if the local remote still points at
  the old name.

Net: the repo rename is low-risk and reversible in effect (redirects), unlike
the machine rename.

---

## 5. Name-availability check (before committing)

For each proposed name, confirm it doesn't collide with:
- an existing Buzz/ReBuzz machine (browser display name),
- a `thepedal` repo,
- a well-known commercial plugin/synth (to avoid trading one trademark issue
  for another).

The §1 proposals are generic words chosen to minimise this, but verify per name.

---

## 6. Post-rename housekeeping

- **Roster** re-pulls names from the GitHub API
  (`api.github.com/users/thepedal/repos`), so a fresh pull picks up new repo
  names automatically; old repo-name references elsewhere in the notes should be
  updated (URLs still redirect, but read stale).
- **About box URL** hard-codes the repo path — update it to the new URL for
  cleanliness (the machine is being rebuilt anyway).
- **Licensing §6** open items: tick off the rename decision per machine as it's
  done.

---

## 7. Worked order for one machine (checklist)

1. Pick + availability-check the name (§1, §5).
2. Rename the GitHub repo (§4); update local remote.
3. Edit source: `AssemblyName`, `MachineDecl.Name`/`ShortName` (§3.1).
4. Rename `.prs.xml`, fix its `Machine="…"` attribute, update deploy target +
   `gen_presets.py` (§3.1 items 3-6).
5. Update About box (name/title/URL), README, SESSION_STATE (§3.1 items 7-8).
6. Build; confirm the machine appears under the new browser name and presets
   auto-load.
7. Update the Roster row (or re-pull) (§6).
8. For existing songs: swap the instance (peer-controlled = quick) or keep the
   old DLL alongside (§3.2).
