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

## Results

### A) With vs without any type-aware work

`oxlint -c <config> src/`

| Command | Mean [s] | Min [s] | Max [s] | Relative |
|:---|---:|---:|---:|---:|
| WITHOUT `no-unnecessary-type-parameters` | 16.31 ± 1.09 | 15.66 | 17.57 | 1.00 |
| WITH    `no-unnecessary-type-parameters` | 45.20 ± 0.96 | 44.10 | 45.84 | 2.77 ± 0.19 |

Enabling the rule adds **~28.9 s** (≈ **2.77x slower**) over a run with no
type-aware rules at all.

### B) Incremental cost (tsgolint already invoked by another type-aware rule)

| Command | Mean [s] | Min [s] | Max [s] | Relative |
|:---|---:|---:|---:|---:|
| `await-thenable` only | 17.01 ± 1.11 | 15.74 | 17.76 | 1.00 |
| `await-thenable` + `no-unnecessary-type-parameters` | 44.21 ± 0.68 | 43.69 | 44.99 | 2.60 ± 0.17 |

Adding the rule on top of an already-running tsgolint pass costs essentially
the same — about **+27.2 s** — confirming the cost is the rule itself, not
tsgolint startup. `await-thenable` adds <1 s over the baseline.

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
