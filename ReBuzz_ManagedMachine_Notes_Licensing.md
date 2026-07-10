# ReBuzz Managed Machine Notes — Licensing & Naming

Source: licensing/trademark review done during Pedal Juno106 v1.8 (About-box
URL + GPL-3.0 + LICENSE file). Findings verified 2026-07-11 against the public
GitHub repos and web sources noted below — treat the *facts* as a snapshot
(ReBuzz could add a license later; check again before relying on them).

**This is engineering notes, not legal advice.** None of the authors here are
lawyers. For anything load-bearing (a release you're distributing widely, a
commercial move, a cease-and-desist), get an IP attorney. The point of this
file is to record what we found and the sensible defaults we adopted, so we
don't re-derive them per machine.

Sections numbered locally. `Build §N` → `ReBuzz_ManagedMachine_Notes_Build.md`,
`Roster` → `ReBuzz_ManagedMachine_Notes_Roster.md`.

---

## 1. The host has no license — what that means for us

- **ReBuzz** (`github.com/wasteddesign/ReBuzz`) was open-sourced in Sep 2024,
  but there is **no `LICENSE`/`COPYING` file** in the repo and no license
  statement in its README (checked root + common subdirs, `main` branch). The
  `wasteddesign/BuzzMachineExamples` repo is the same — no license file.
- ReBuzz is built on **Jeskola Buzz**, which is **freeware** (gratis,
  closed-source); parts of the Buzz code are described by the community as
  still secret.
- Consequently the interface DLLs our machines compile against —
  `BuzzGUI.Interfaces`, `BuzzGUI.Common`, `Buzz.MachineInterface` — are WDE's
  copyrighted code that is, by default (absent an explicit grant),
  **all-rights-reserved**. The ecosystem operates on clear *implied*
  permission to write machines (WDE ships examples and invites machine
  development), not a written licence.

### 1.1 Why this is low-risk for us specifically

Our machine packages ship **only our own code**. Each `.csproj` references the
host DLLs with `<Private>false</Private>` (Build §1 references), so they are
**never copied into build output and never redistributed** — the user already
has ReBuzz installed, and the linking happens on their machine at load time.
Redistribution is what a licence would govern, and we don't redistribute WDE's
code. So the missing ReBuzz licence is WDE's gap to close, not a distribution
problem for us.

**Keep it that way:** never bundle a ReBuzz/Buzz DLL into a machine's release
zip or gear folder. `Private=false` on every host reference; ship dll +
`.prs.xml` + source + LICENSE only.

---

## 2. GPL-3.0 on our own code — fine, plus a linking exception

We can licence our own original DSP however we like; **GPL-3.0 is valid** (used
from pedal-juno106 v1.8 onward). The one genuinely murky spot is the classic
"GPL program dynamically linking a non-GPL host" question. Because we
distribute only our code and the *user* performs the linking at runtime, the
practical risk is low — but the airtight fix is a **GPL v3 §7 additional
permission (linking exception)** that explicitly allows combining our machine
with ReBuzz/Buzz and its interface libraries, whatever their licence terms.

### 2.1 Linking-exception template

Add near the top of the machine's primary source file (and/or reference it
from the `LICENSE`). Review with counsel before relying on it; this is a
sensible default, not vetted legal text:

```
Additional permission under GNU GPL version 3 section 7:

As a special exception, the copyright holders of this program give you
permission to combine this program with the ReBuzz and Jeskola Buzz host
applications and their interface libraries (including, without limitation,
BuzzGUI.Interfaces, BuzzGUI.Common, and Buzz.MachineInterface), and to
convey the resulting combined work. This permission applies regardless of
the licence terms (or absence thereof) under which those host components
are made available. You must still comply with the GNU GPL version 3 in
all respects for all of the program's own code.
```

GPL-3.0 is also **inbound-compatible with MIT/BSD** permissive code — you can
incorporate MIT/BSD-licensed source into a GPL-3.0 work as long as you retain
the original notice (relevant to the ported machines, §4).

---

## 3. Trademark — descriptive use vs. product-name use

The distinctive part of a name is what matters; the `Pedal` prefix shields
nothing. Two uses with very different risk:

- **Descriptive / nominative** — "an emulation of the Roland Juno-106" *in the
  description* to say what the machine does. More defensible: you're allowed to
  name the thing you emulate, provided you use no more of the mark than needed
  and don't imply the manufacturer made or endorsed it.
- **As the product's own name** — e.g. `Pedal Juno106`, short-name `Juno106`,
  using the model name as the source identifier for *our* product. This is the
  higher-risk use (confusion / implied affiliation).

### 3.1 The adopted mitigations (from Juno v1.8)

1. **Disclaimer** in the About box and README:
   *"Roland and Juno-106 are trademarks of Roland Corporation. This project is
   independent and is not affiliated with, authorised by, or endorsed by
   Roland. The names are used only to describe the hardware this machine
   emulates."* Undercuts implied endorsement; does **not** cure using the model
   name as the product name.
2. **The rename template** (the low-risk pattern already used by two machines):
   a **functional or punny original name** + "emulates the X" in the
   description + the disclaimer. Follow `pedal-Faze-R` (phase-distortion pun; no
   "Casio"/"CZ" in the name) and `pedal-invFFT` (K5000-inspired; no "Kawai").

### 3.2 Per-machine sweep

| Machine | Mark in the *name* | Holder | Risk | Note |
|---|---|---|---|---|
| pedal-juno106 | Juno-106 | Roland | High | model name as product name; Roland enforces marks |
| pedal-sh101   | SH-101   | Roland | High | same |
| pedal-M1      | M1       | Korg   | High | same |
| pedal-dly-PCM41 | PCM 41 | Lexicon (Harman) | Med-High | model number as name; desc says "PCM41-style" |
| pedal-zplane  | Z-Plane  | E-mu / Creative | Med | "Z-Plane" is an E-mu term |
| pedal-plaits  | Plaits   | Mutable Instruments | Low-Med | see §4 — open-source, defunct vendor, low enforcement |
| pedal-Faze-R  | — (pun)  | (Casio CZ lineage) | Low | **template to copy** |
| pedal-invFFT  | — (functional) | (Kawai K5000 heritage) | Low | **template to copy** |
| pedal-profiler / profiler2 | "Profiler" (generic) | (cf. Kemper) | Low | generic computing term; these are CPU tools, not amp sims |
| pedal-tracker | "Matilde" (in desc only) | community | Low | Matilde is a classic Buzz machine, not a commercial mark |
| all others (comp, eq, gain, chorus, gate, filter, folder, plate, hallverb, …) | — | — | None | generic/descriptive names |

If we rename, the **five High/Med rows** are the candidates. Keeping them is a
conscious-risk choice, not a safe default — Roland especially.

---

## 4. The "Port of X" machines — a separate, more concrete axis

Porting another author's machine implicates **their** copyright/licence — a
different (and often more actionable) issue than trademark. Audit these and
make sure the required attribution is actually present in the shipped machine:

- **pedal-plaits** — Mutable Instruments' firmware is **MIT**-licensed and
  everything was open-sourced when Mutable wound down, so porting the DSP is
  *explicitly permitted* — **but MIT requires retaining Mutable's copyright +
  licence notice** in the source/distribution. Confirm it's there. (MIT →
  GPL-3.0 incorporation is fine; keep the MIT block.) This also softens the
  §3.2 trademark note: the project was deliberately open and the vendor is
  defunct.
- **BTDSys-PeerCtrl-ReBuzz** ("port of BTDSys PeerCtrl") and **pedal-retrig**
  ("port of 'genre' by intoxicated") — ports of other Buzz authors' machines.
  Confirm each original's licence, or that we have the author's permission. If
  a source is unlicensed, we're relying on implied permission — decide
  consciously whether we're comfortable, and credit the original author either
  way.
- **pedal-tracker** ("Matilde-compatible") — the Tracker addendum references a
  **BSD** mirror of Matilde (`github.com/Buzztrax/buzzmachines/Matilde`). If
  any of it was reused, keep the BSD notice; if it's clean-room behaviour, fine
  — but record which it is.

**Rule going forward:** any machine described as a "port of" or "compatible
with" another author's work gets a `## Credits / third-party licences` section
in its README naming the origin and its licence, and carries the upstream
notice file if code was reused.

---

## 5. Checklist for a new (or audited) machine

1. **Host DLLs** referenced with `Private=false`; never bundled/redistributed.
2. **Own licence** chosen (GPL-3.0 is the house default from Juno v1.8); ship a
   `LICENSE` file with the full text (Build §3 doesn't deploy it — repo-root
   only). GPL text fetched verbatim from a canonical source (SPDX), not typed.
3. **Linking exception** (§2.1) added if GPL and you want the host-linking
   question closed.
4. **Trademark**: prefer an original/functional name; if the name carries a
   third-party mark (§3.2), add the disclaimer and treat a rename as the
   cleaner fix.
5. **Ports**: upstream licence confirmed, attribution/notice present (§4).
6. **About box** (AboutWindow §1.4): name, version, description, author, URL,
   licence, and — for emulations — the trademark disclaimer.

---

## 6. Open items (as of 2026-07-11)

- ReBuzz has no declared licence — periodically re-check the repo; if WDE adds
  one, revisit §2's GPL-compatibility reasoning.
- Five machines (§3.2 High/Med) carry model-name trademarks — rename decision
  outstanding.
- Ported-machine attributions (§4) not yet audited machine-by-machine.
