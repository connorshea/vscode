# Oxlint + oxlint-tsgolint benchmark: `typescript/no-unnecessary-type-parameters`

Measures the runtime impact of enabling
`typescript/no-unnecessary-type-parameters` when running Oxlint with the
type-aware plugin (`oxlint-tsgolint`) on the `vscode` source tree.

## Setup

- Repo: `microsoft/vscode` (this checkout), linting `src/` (6,224 files)
- Oxlint: `1.65.0`
- oxlint-tsgolint: `0.22.1`
- Node: `v22.22.2`
- Threads: 4 (oxlint default for this machine)
- Benchmark tool: `hyperfine 1.20.0`, 1 warmup + 3 timed runs per config
- All configs have `options.typeAware: true`, the `typescript` plugin, and the
  same baseline non-type-aware rules (`eqeqeq`, `no-debugger`, `no-unused-vars: off`).
- Configs used:
  - `.oxlintrc.benchmark-without.json` — no type-aware rules
  - `.oxlintrc.benchmark-with.json` — adds `typescript/no-unnecessary-type-parameters`
  - `.oxlintrc.benchmark-baseline.json` — one cheap type-aware rule (`typescript/await-thenable`)
  - `.oxlintrc.benchmark-baseline-plus.json` — `await-thenable` + `no-unnecessary-type-parameters`
  - `.oxlintrc.benchmark-dozen.json` — 12 type-aware rules (no `no-unnecessary-type-parameters`):
    `await-thenable`, `no-floating-promises`, `no-misused-promises`,
    `no-base-to-string`, `no-for-in-array`, `no-implied-eval`,
    `no-redundant-type-constituents`, `no-unnecessary-type-assertion`,
    `no-unsafe-argument`, `restrict-plus-operands`,
    `restrict-template-expressions`, `require-await`

## Results

`oxlint -c <config> src/`, all configs with `options.typeAware: true`. The
"Relative" column is normalized to the fastest config.

| Config | Mean [s] | Min [s] | Max [s] | Relative |
|:---|---:|---:|---:|---:|
| no type-aware rules                                  | 16.31 ± 1.09 | 15.66 | 17.57 | 1.00x |
| `await-thenable` only                                | 17.01 ± 1.11 | 15.74 | 17.76 | 1.04x |
| 12 type-aware rules (no `no-unnecessary-type-parameters`) | 26.51 ± 1.12 | 25.76 | 27.79 | 1.63x |
| `no-unnecessary-type-parameters` only                | 45.20 ± 0.96 | 44.10 | 45.84 | 2.77x |
| `await-thenable` + `no-unnecessary-type-parameters`  | 44.21 ± 0.68 | 43.69 | 44.99 | 2.71x |

- Enabling `no-unnecessary-type-parameters` adds **~28.9 s** (≈ **2.77x slower**)
  over a run with no type-aware rules.
- `await-thenable` alone adds <1 s — confirming tsgolint startup is cheap and
  most type-aware rules are not the dominant cost.
- Going from 1 to 12 type-aware rules adds **~8.9 s** (≈ **0.8 s per extra rule**
  on average), still ~20 s less than enabling `no-unnecessary-type-parameters`
  on its own.
- Adding `no-unnecessary-type-parameters` on top of `await-thenable` costs
  about the same **+27 s**, so the cost is the rule itself, not tsgolint
  startup.

### Errors reported on `src/`

- WITHOUT the rule: 43 errors
- WITH    the rule: 243 errors → **200 `no-unnecessary-type-parameters` violations** in `src/`

## Takeaway

`typescript/no-unnecessary-type-parameters` is the dominant cost in this
benchmark by a wide margin. On the vscode `src/` tree it roughly **triples**
total lint time (16 s → 45 s), independent of whether tsgolint is already
warm from another type-aware rule. If lint latency matters, consider scoping
this rule to a smaller `overrides` glob, gating it to CI, or reviewing/silencing
the 200 existing violations and deciding case-by-case whether the value
justifies the runtime cost in interactive runs.
