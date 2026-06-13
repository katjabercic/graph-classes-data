# Graph-classes seed dataset

A tiny, self-contained pi-base data repository for the **graph classes** domain, in the
exact Markdown/YAML format the compiler expects (`properties/`, `spaces/`, `theorems/`,
`spaces/<id>/properties/`). It exists to demonstrate that pi-base's object/property/theorem
model and its deduction engine work unchanged for graph classes — see
[../meeting-prep.md](../meeting-prep.md) for the conceptual mapping.

The mapping used here:

| pi-base | here |
|---|---|
| `Property` | a graph **class** |
| `Theorem` (implication) | a class **inclusion** |
| `Space` (object) | a specific **example graph** |
| `Trait` | "graph *G* is in class *C*" |
| counterexample | a graph in one class but not another |

## Contents

**Classes (`properties/`):** bipartite (P1), planar (P2), perfect (P3), chordal (P4),
forest (P5), complete (P6).

**Inclusions (`theorems/`):**
- T1 forest ⇒ bipartite
- T2 forest ⇒ planar
- T3 bipartite ⇒ perfect
- T4 chordal ⇒ perfect

**Example graphs (`spaces/`):** K₅ (S1), C₅ (S2), C₄ (S3), claw K₁,₃ (S4).

## Result (compiled and verified with the deduction engine)

Only the **bold** cells are authored as traits; the rest are what the viewer's
forward-chaining + contrapositive prover ([`packages/core/src/Logic/Prover.ts`](../packages/core/src/Logic/Prover.ts))
derives. This has been confirmed: the repo compiles with no contradictions, and running
`deduceTraits` per space produces exactly the closure below (the claw's *perfect* comes
from the chained proof T1 → T3).

| graph | bipartite | planar | perfect | chordal | forest | complete |
|---|---|---|---|---|---|---|
| **K₅** | **F** | **F** | T *(T4: chordal⇒perfect)* | **T** | **F** | **T** |
| **C₅** | **F** | **T** | **F** ← imperfect witness | **F** | **F** | **F** |
| **C₄** | **T** | **T** | T *(T3: bipartite⇒perfect)* | **F** | **F** | **F** |
| **claw** | T *(T1)* | T *(T2)* | T *(T3, chained from T1)* | *(unknown)* | **T** | **F** |

Two things to point at in the meeting:
- **The claw** authors a single positive trait (`forest = true`) and the engine derives
  *bipartite → planar → perfect* — the lattice closure in action.
- **C₅** is the classic odd hole (χ=3 > ω=2): it carries neither defining property of
  *perfect*, so nothing forces perfection and its authored `perfect = false` stands. It is
  the witness that *perfect* is strictly smaller than the classes above it. C₄ being
  bipartite-but-not-chordal, and K₅ being chordal-but-not-bipartite, witness that those two
  classes are incomparable.

## Reproduce the bundle (project-level, no global installs)

`graph-classes-seed/bundle.json` is already compiled and verified. To rebuild it from a
clean checkout:

```bash
# from the repo root — all installs stay in ./node_modules
corepack enable          # provides pnpm without a global install
pnpm install             # populates ./node_modules; the "Ignored build scripts" notice is harmless

# Build @pi-base/core (peggy generates the parser, then tsc emits dist/esm + dist/cjs).
# These call the binaries directly: pnpm 11's pre-run deps check otherwise aborts on the
# un-approved build scripts above — run `pnpm approve-builds` once if you prefer `pnpm run`.
cd packages/core
./node_modules/.bin/peggy --plugin ./node_modules/ts-pegjs/dist/tspegjs -o src/Formula/Grammar.ts --cache src/Formula/Grammar.pegjs
./node_modules/.bin/tsc --module es2022   --outDir dist/esm && printf '{"type": "module"}'    > dist/esm/package.json
./node_modules/.bin/tsc --module commonjs --outDir dist/cjs && printf '{"type": "commonjs"}' > dist/cjs/package.json
cd ../..

# Compile this seed repo → graph-classes-seed/bundle.json. The GITHUB_* vars stand in for
# git version metadata, which the compiler would otherwise read from the data repo's .git.
cd packages/compile
GITHUB_REF=refs/heads/main GITHUB_SHA=seed-demo ./node_modules/.bin/tsx src/main.ts ../../graph-classes-seed bundle.json
cd ../..
```

A successful run writes `graph-classes-seed/bundle.json` and exits 0. Any schema or
contradiction error is printed as `::error file=…` and exits 1.

## View it in the viewer

The viewer loads a local bundle when `VITE_BUNDLE_HOST` contains `localhost`: it serves
`packages/viewer/public/refs/heads/main.json` ([`+layout.server.ts`](../packages/viewer/src/routes/+layout.server.ts)).
That path is currently a symlink to the Cypress fixture, so repoint it at the seed bundle:

```bash
cd packages/viewer/public/refs/heads
rm main.json                                   # it's a symlink to the topology fixture
cp ../../../../../graph-classes-seed/bundle.json main.json

# then, from packages/viewer
VITE_BUNDLE_HOST=http://localhost pnpm run dev
```

Open the dev server, go to the spaces (graphs) listing, and open the claw to see the derived
traits with their proofs. Restore the symlink afterwards with `git checkout main.json`.

## Conventions for contributors

When this directory graduates into its own data repository (the home for Anthony's
exported YAML), contributors author plain Markdown + YAML frontmatter, one entity per file:

| Entity | Path | Required frontmatter | Body |
|---|---|---|---|
| Class (property) | `properties/P######.md` | `uid`, `name` | definition |
| Example graph (space) | `spaces/S######/README.md` | `uid`, `name` | description |
| Inclusion (theorem) | `theorems/T######.md` | `uid`, `if`, `then` | justification |
| Membership (trait) | `spaces/S######/properties/P######.md` | `space`, `property`, `value` | justification |

Rules the compiler enforces — worth stating up front so Anthony's export lines up:

- **IDs** are a letter prefix (`S`/`P`/`T`) + a zero-padded integer (`P000003`). They are
  global and unnamespaced, so this repo owns its own numbering — keep a disjoint range from
  topology/topos if the data ever has to coexist. The filename must match the `uid`.
- **Traits are boolean.** `value: true` / `value: false`; *unknown* is expressed by simply
  omitting the trait file, not a third value.
- **Frontmatter is strict.** Any key the schema doesn't recognise fails the build — no ad-hoc
  metadata fields without a code change in `@pi-base/core`.
- **References** use a closed set of source types (`doi`, `wikipedia`, `mr`, `mathse`, `mo`,
  `zb`). A graph-specific source (ISGCI, OEIS) needs a one-line addition to `Ref.ts` upstream
  — a good item for the James Dabbs meeting.
- **Theorems** accept the simple map form (`if: {P000005: true}`) or the richer formula
  syntax (`~` not, `+` and, `|` or).

## Continuous validation

[`.github/workflows/validate.yaml`](.github/workflows/validate.yaml) compiles the whole repo
on every push and fails the job on any schema error, broken cross-reference, or contradiction
— the same checks we ran by hand, automated. It activates once this directory is a standalone
repository (the workflow file has to sit at the repo root to run).

## Notes / known seams for a real fork
- IDs are global integers with no namespace; a real graph-classes repo owns its own numbering.
- The reference kinds are a closed enum ([`Ref.ts`](../packages/core/src/Ref.ts)); adding a
  graph source like ISGCI or OEIS needs a code change, not just data.
- Numeric invariants (chromatic number, treewidth) have no native representation — see the
  limitation section in [../meeting-prep.md](../meeting-prep.md).
