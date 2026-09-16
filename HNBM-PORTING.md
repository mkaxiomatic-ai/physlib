# Porting HNBM into Physlib

**This file exists only on the fork.** It is the working record for moving material from
[mkaratarakis/HNBM](https://github.com/mkaratarakis/HNBM) into
[leanprover-community/physlib](https://github.com/leanprover-community/physlib), one PR at a time.
It should never be part of a PR to upstream.

## 1. What this branch is

`hnbm-import` is `upstream/master` with HNBM grafted on top, then repaired until it compiles
against physlib's toolchain. The graft is a real merge of HNBM's history
(`--allow-unrelated-histories`), so `git log --follow` and `git blame` still reach the original
commits. That matters: physlib's [AI-POLICY.md](AI-POLICY.md) §1.7 makes a human responsible for
vouching that every definition and proof means what it claims, and blame is how you find who
wrote what.

The import is **purely additive**. No file that upstream owns is modified except `lakefile.toml`,
which gains the three HNBM libraries as *non-default* targets — a bare `lake build` still builds
exactly what upstream builds.

## 2. Provenance

| | |
|---|---|
| Source | `mkaratarakis/HNBM` @ `a9a7755008a756ca416b304100e44d52bc203ef6` |
| Source toolchain | `leanprover/lean4:v4.30.0`, Mathlib `v4.30.0` |
| Target toolchain | `leanprover/lean4:v4.33.0`, Mathlib `v4.33.0` |
| Size | ~25k lines of Lean across 100 files |

Dropped at import: HNBM's `lakefile.lean`, `docbuild/`, its blueprint CI workflow, and three
`.DS_Store` files. `NeuralNetwork.lean` is kept but **not** a build target — its three imports do
not exist at this commit, so it is dead at the pinned revision.

## 3. Build status

All three libraries build clean on Lean 4.33 / Mathlib v4.33.

| Library | Status | Notes |
|---|---|---|
| `PF` | green | Perron–Frobenius theory. No `sorry`. |
| `MCMC` | green | 22 pre-existing `sorry`s, all in `MCMC/Gibbs.lean`. |
| `HopfieldNet` | green | No `sorry`. |

No `axiom` is declared anywhere in the import, and the sorry count is unchanged from
HNBM@`a9a7755` — the port added none.

Reproduce with `lake exe cache get` then `lake build PF MCMC HopfieldNet`.

## 4. What the port had to change

Almost all of it is Mathlib drift between v4.30 and v4.33, not mathematics:

- **`mulVec` is now defined through `Matrix.row`/`dotProduct`.** `Matrix.mulVec_apply` gives the
  `M.row i ⬝ᵥ v` form; the old sum form is `Matrix.mulVec_apply_eq_sum`. Most of the diff is that
  rename inside `simp` sets. `dotProduct` lives in the root namespace, not `Matrix`.
- **`simp` no longer zeta-unfolds `let`-bound locals.** Proofs that relied on a local `let` being
  transparent to `simp` now have to name it in the simp set, or use `exact` and lean on defeq.
- **`simp only at h` with no lemmas is an error** when it makes no progress, rather than a no-op.
  Same for `dsimp`.
- **`aux` is a reserved module name** and Lean 4.33 rejects it outright. `{HopfieldNet,
  HopfieldNet.Quiver.BM, PF}.aux` became `.Auxiliary`.
- **`Quiver.inducedQuiver` needed `@[instance_reducible]`** for subquiver arrows to unify.
- Assorted renames: `Module.End.mul_apply`, `maxGenEigenspace` as an `OrderHom` at `⊤`,
  `Function.comp_def` where `Tendsto.comp` used to normalise on its own.

None of these changed a single statement. Where a proof was repaired by `exact` rather than
`simpa`, it is because the two sides were already definitionally equal.

## 5. Where the material should land

Physlib splits into `Physlib` (high review bar) and `PhyslibAlpha` (arXiv-style "one look"
review, file structure mirroring `Physlib`). **Everything here should target `PhyslibAlpha`
first.** Nothing in this import has been through physlib's review standard.

Proposed destinations, subject to maintainer preference:

| From | To | Rationale |
|---|---|---|
| `PF/PerronFrobenius/*` | `PhyslibAlpha/Mathematics/LinearAlgebra/PerronFrobenius/` | General matrix theory. Arguably Mathlib material rather than physlib's; worth asking upstream first. |
| `PF/Combinatorics/Quiver/*`, `PF/Data/List.lean` | `PhyslibAlpha/Mathematics/` | Supporting combinatorics. |
| `MCMC/*` | `PhyslibAlpha/Mathematics/Probability/` | Markov chain convergence, total variation, detailed balance. |
| `HopfieldNet/Quiver/NeuralNetwork/toCanonicalEnsemble.lean` | `PhyslibAlpha/StatisticalMechanics/` | **Best first PR.** It already bridges to physlib's own `CanonicalEnsemble`. |
| `HopfieldNet/Quiver/BM/*` | `PhyslibAlpha/StatisticalMechanics/BoltzmannMachine/` | The Boltzmann machine is statistical mechanics proper. |
| `HopfieldNet/Quiver/HN/*` | `PhyslibAlpha/StatisticalMechanics/HopfieldNetwork/` | |

Upstream has **no** overlapping material — no Hopfield, no Perron–Frobenius, no MCMC machinery.
The only contact point is `Physlib.StatisticalMechanics.CanonicalEnsemble`, which HNBM already
builds on rather than duplicates.

## 6. Gates before anything is PR'd

Physlib's [AGENTS.md](AGENTS.md) is a hard contract. Known blockers in the current import:

- [ ] **Module system.** All 583 `Physlib/` files use `module` / `public import` /
      `@[expose] public section`. No HNBM file does. A non-module file *can* import a module file
      (verified), which is why the fork builds — but anything landing in `PhyslibAlpha` must be
      converted.
- [ ] **`sorry` is banned.** `MCMC/Gibbs.lean` has 22. That file cannot be PR'd until they are
      discharged, or it must be split so the sorry-free part goes first.
- [ ] **`∃ x, ..., True` is banned.** `PF/PerronFrobenius/Aperiodic.lean:71`
      (`exists_frobenius_normal_form`) is exactly this shape — it states the Frobenius normal
      form and then proves nothing. It must be given real content or dropped.
- [ ] **`lemma` not `theorem`** unless the result is well known in the physics literature. The
      import has 232 `theorem`s against 753 `lemma`s; most of those `theorem`s need demoting.
- [ ] **Docstrings on every definition**, and section headers numbered `# A.`, `## A.1.`.
- [ ] **Copyright headers.** 17 files have none. The rest say "Released under Apache 2.0",
      authored by Matteo Cipollina.
- [ ] **PR scope: one coherent concept per PR.** The tables in §5 are far too coarse to be PRs;
      each will need splitting.
- [ ] `lake exe lint_all`, `./scripts/lint-style.sh`, and for alpha files
      `lake exe runPhyslibAlphaLinters`, `lake exe noAlphaImports`, `lake exe alphaFileImports`.

## 7. Open questions for the author

1. **Licence.** HNBM's root `LICENSE.md` is **MIT**; physlib is **Apache 2.0**, and HNBM's own
   file headers already say Apache 2.0. MIT is compatible with relicensing to Apache 2.0, but the
   copyright holder has to say so. The root `LICENSE.md` was imported unchanged and is currently
   the odd one out in an Apache repo.
2. **Authorship.** The file headers name Matteo Cipollina. AI-POLICY §1.7 and §2.1 require a named
   human to vouch for each contribution and to verify every bibliographic reference personally.
3. **Should `PF` and `MCMC` go to Mathlib instead?** Perron–Frobenius and Markov chain convergence
   are general mathematics. Physlib accepts supporting material under `Mathematics/`, but upstream
   may prefer these land in Mathlib.

## 8. Import audit — what came across, and what did not

Verified by comparing the branch against `a9a7755` directly.

### Declarations: nothing lost

Extracting every `theorem`/`lemma`/`def`/`abbrev`/`instance`/`structure`/`class`/`inductive`/
`opaque` from the HNBM-side Lean files:

| | |
|---|---|
| HNBM @ `a9a7755` | 1526 |
| this branch | 1527 |

The entire difference is three deliberate edits:

- `def State'` → `abbrev State'` (`HN/Core.lean`) — same declaration, keyword only.
- `instance IsHamiltonian_of_EnergySpec'` → `def` (`toCanonicalEnsemble.lean`) — same declaration,
  keyword only; it can no longer be an `instance` because `EnergySpec'` is a structure.
- `+ lemma distributionPushforward_eq` (`MCMC/Convergence.lean`) — a new `rfl`-lemma, added
  because the definition no longer unfolds under `simp`.

**No declaration was removed or renamed.**

### Statements: unchanged

The port touches 36 files, 138 insertions and 110 deletions. Every removed line is a tactic
(`simp`, `simpa`, `rw`, `dsimp`, `grind`, `conv`), an `import` updated for the `aux` → `Auxiliary`
rename, or one of the two keyword changes above. No theorem, lemma or definition *statement* was
altered. Anything not in that diff is byte-identical to `a9a7755`.

### Files not carried over

All are build or editor metadata; none contains Lean content. Every one is still reachable in this
repository — the HNBM history is merged in, so `git show a9a7755:<path>` recovers any of them.

| Path | Why |
|---|---|
| `lakefile.lean`, `lakefile.lean.bak` | superseded by the fork's `lakefile.toml` |
| `docbuild/{lakefile.toml,lake-manifest.json,lean-toolchain}` | doc-gen scaffolding for the standalone repo |
| `.github/workflows/blueprint.yml` | HNBM's blueprint CI; it drives `leanblueprint` over a `blueprint/` directory that **does not exist** at `a9a7755`, so it could not have run |
| `HopfieldNet/.DS_Store`, `PF/.DS_Store`, `PF/Combinatorics/.DS_Store` | macOS Finder junk |

Renamed, content preserved: `{HopfieldNet,HopfieldNet/Quiver/BM,PF}/aux.lean` → `Auxiliary.lean`.
Two are byte-identical; `HopfieldNet/Auxiliary.lean` additionally carries one proof repair.

`NeuralNetwork.lean` **is** kept, even though it is dead at this revision and is not a build target.

### Files overwritten by physlib's versions

These paths exist in both repositories and the merge kept physlib's. HNBM's versions are build
metadata pinned to Lean v4.30 (`lean-toolchain`, `lake-manifest.json`), editor config
(`.vscode/settings.json`), a two-line `.gitignore`, and `Physlib.lean` — a nine-line entry point
importing three `Physlib.*` modules, which is meaningless now that this *is* physlib.

The exception worth recording is HNBM's `README.md`, whose whole text was:

> This repository contains the source code of a paper submitted to the Arxiv. The formalization is
> built using **Lean 4 (v4.28.0-rc1)** and is compatible with **Mathlib** version `e318a59`
> (February 20, 2026). [...] The code is provided as a `.zip` file. Extract the contents and
> navigate to the project's root directory in your terminal. Run `lake build` to check that the
> project builds correctly.

Note it claims Lean v4.28.0-rc1 while `a9a7755`'s `lean-toolchain` pins v4.30.0 — the README was
already stale. The arXiv submission it refers to is not identified; that link needs to be
established by hand before any PR cites it.

`LICENSE.md` (MIT) was imported unchanged — see §7.1.
