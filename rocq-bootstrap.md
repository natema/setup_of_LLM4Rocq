# Bootstrapping a Rocq formalization of a paper — setup guide for a coding agent

You have been started in a folder containing the LaTeX sources of a paper, and the goal
is to formalize its results in Rocq and possibly other libraries like MathComp. This document advises how to do that. 

**Copy this file into the new project** (as `docs/BOOTSTRAP.md`, say) and, once you have
worked through part 4, write the project's own `AGENTS.md`/`CLAUDE.md` from part 5.

Behind this are two surveys done on 2026-09-10/12: one of the LLM4Rocq tooling
(pytanque → rocq-mcp → the two Claude Code plugins, plus rocqet-search) and one of the
org's dozen formalization repos (layout, build, CI, verification gates). Everything
load-bearing from them is reproduced here; appendix A keeps the per-repo detail that
parts 1–8 compress, so this file stands alone. Pins drift — re-check them (part 1.3)
before trusting a version number here.

---

## 0. The one rule

**The paper is the specification. Statements are never weakened to make them provable.**

Not the constants, not the exponents, not the hypotheses. If a statement in the paper
looks wrong or unprovable, stop and record it in `docs/PLAN.md` — do not quietly repair
it. A formalization of a theorem you adjusted proves nothing about the paper.

Corollary: keep the LaTeX source in the repo (read-only) and make every Rocq declaration
cite the paper's numbering (`Lemma 3.4`) in its docstring. You will need this constantly,
and a reviewer needs it to trust the result.

---

## 1. Toolchain

### 1.1 Why these versions

Rocq **9.1.1**, MathComp **2.5.0**, mathcomp-classical/reals/analysis **1.16.0**,
hierarchy-builder **1.10.2**, algebra-tactics **1.2.7**, zify, coq-lsp **0.2.5+9.1**,
OCaml 5.2.1.

Rocq stays **below 9.2** on purpose: the released coq-lsp (which ships `pet`/`pet-server`,
the petanque backend every interactive agent tool uses) requires `rocq-core >= 9.1 < 9.2`.
Upgrading Rocq past 9.1 silently costs you the interactive tooling. Check this constraint
before choosing a newer Rocq.

### 1.2 Install (user level, one switch, never ad hoc)

```bash
opam switch create rocq-9.1.1 ocaml-base-compiler.5.2.1   # if it does not exist
eval $(opam env --switch=rocq-9.1.1 --set-switch)
opam repo add rocq-released https://rocq-prover.org/opam/released
opam install \
  rocq-core.9.1.1 coq-core.9.1.1 rocq-stdlib.9.1.0 \
  rocq-mathcomp-{ssreflect,fingroup,algebra,solvable,field}.2.5.0 \
  rocq-mathcomp-{classical,reals,analysis}.1.16.0 \
  rocq-mathcomp-finmap rocq-mathcomp-bigenough \
  rocq-hierarchy-builder.1.10.2 coq-mathcomp-algebra-tactics.1.2.7 \
  coq-mathcomp-zify coq-lsp.0.2.5+9.1
```

Python side (uv, Python ≥ 3.11):

```bash
uv tool install "git+https://github.com/LLM4Rocq/rocq-mcp"
uv tool install "rocqet[mcp] @ git+https://github.com/LLM4Rocq/rocqet-search"
uv tool install rocqblueprint          # only if you want a blueprint (part 6)
uv tool install plastex                # rocqblueprint web shells out to a `plastex` binary
```

Agent tooling (Claude Code; adapt for other CLIs):

```bash
claude mcp add -s user rocq-mcp -e ROCQ_COQC_TIMEOUT=120 -e ROCQ_VERIFY_TIMEOUT=240 \
  -- ~/.local/bin/opam exec --switch=rocq-9.1.1 -- ~/.local/bin/rocq-mcp
claude mcp add -s user rocqet -e ROCQET_API_URL=https://rocqet-api.onrender.com \
  -- ~/.local/bin/rocqet-mcp
claude plugin marketplace add LLM4Rocq/rocq-skills && claude plugin install rocq@rocq-skills
# mathcomp-skills ships no marketplace.json: wrap it in a local marketplace dir first.
```

Install these at **user level, not per project** — you will start many formalization
repos and they are all the same toolchain.

### 1.3 Verify before believing anything

```
rocq_health                      # which switch and binaries the MCP server actually sees
rocq_query(preamble="From mathcomp Require Import all_ssreflect all_algebra.",
           command="Check addnC.")
rocqet_search("exponential bound")
```

`rocq_health` is the one that catches the most common failure: the MCP server inherits
`PATH` from whatever launched the CLI, so it can silently be on the wrong switch. That is
why the `claude mcp add` line above pins it with `opam exec --switch=…`.

### 1.4 Install traps (all verified, all still bite)

1. `uvx rocq-mcp` / `pipx install rocq-mcp` **do not work** — it is not on PyPI. Install from git.
2. `pip install pytanque` installs an **unrelated package**. Use `git+https://github.com/LLM4Rocq/pytanque@v0.2.2`.
3. The MCP server must be named **exactly** `rocq-mcp`: both plugins hard-code `mcp__rocq-mcp__rocq_*`.
4. On Rocq 9, `coqc` comes from the `coq-core` compat package (or set `ROCQ_COQC_BINARY`).
5. New `.claude/agents/*.md` are not selectable until the CLI restarts.
6. `rocqblueprint web` shells out to a bare `plastex`; the one inside rocqblueprint's own
   tool venv is not on `PATH`, so install `plastex` as its own tool or the web build fails
   with `plastex: command not found`.
7. The `rocq` plugin's PreToolUse hook blocks `git push`, `commit --amend`, `gh pr create`
   anywhere `.v` files exist, and always blocks `reset --hard`, `clean -f`, `checkout --`,
   `restore`. Lift the first group with `ROCQ_GUARDRAILS_COLLAB_POLICY=allow`.
   **It matches the command text, not the command**: a commit message or heredoc
   containing the words "git push" or "git restore-mtime" trips it. Write such text to a
   file with your editing tool and use `git commit -F file`.

---

## 2. Read the paper before writing any Rocq

Produce, in `docs/DESIGN.md`, *before* the first `.v` file:

- **The statement list**: every definition, lemma, theorem you intend to formalize, with
  the paper's numbering, in dependency order. This is the project plan.
- **Representation decisions**, with justification. This is where formalizations succeed
  or fail. For each object in the paper: what Rocq type? A finite probability space as
  `\sum_` bigops over a `finType` divided by cardinalities, or measure-theoretic
  `probability T R`? `{set 'I_n}` or a predicate? `seq` or tuple? Decide once, write it
  down, and do not drift.
- **A dictionary**: paper notation → Rocq name. Append to it every single time you find a
  library lemma; it is the highest-value artifact you produce and it compounds.

If the paper has a companion formalization in another system (Lean/Mathlib), vendor it
read-only as a submodule and map declaration-by-declaration. Porting a proof is *far*
cheaper than reconstructing one, and gives you an oracle for statement faithfulness.

**Prefer a minimal bespoke layer over a heavyweight library** when the paper only needs a
sliver of it. One development needed finite uniform probability and a single Chernoff
bound; building that on bigops was much cheaper than adopting measure theory, and kept the
axiom footprint to the three `boolp` axioms. (Note what the ecosystem does *not* give you:
no repo in the org has a Chernoff or Hoeffding bound, none uses infotheo, and finite
probability is not first-class anywhere — mathcomp-analysis' `probability.v` is
measure-theoretic (`\bar R`, Lebesgue integral) and mathcomp-qbs builds on that. Search it
first with `rocq_query`, but expect to write the finite layer yourself.)

---

## 3. Scaffold the repo

```
theories/prelude.v        shared imports, options, smoke lemmas
theories/<layer>/…        one directory per layer, in dependency order
scripts/check_layers.py   enforces the import layering
scripts/check_admitted.py no Admitted/admit/Axiom/Parameter on main
verify.sh                 clean rebuild + Print Assumptions of the headline theorems
_CoqProject               every .v file, in layer order
Makefile                  thin wrapper over the generated CoqMakefile
CoqMakefile.local         coqdoc flags (part 6)
docs/{DESIGN,PLAN}.md     decisions; milestones + status table
paper/                    the LaTeX source, read-only
```

`_CoqProject` carries the mapping and the warning flags:

```
-R theories <Namespace>
-arg -w -arg -notation-overridden,-ambiguous-paths
```

The `Makefile` is the coq-community pattern: generate `CoqMakefile` from `_CoqProject`
with `rocq makefile` and delegate, plus your own `gate` target. Name the generated file
`CoqMakefile` — the `rocq` plugin's `/rocq:checkpoint` invokes `make -f CoqMakefile`.

**Layering is worth enforcing mechanically from day one.** A script that fails the build
when a file imports above its layer costs 40 lines and prevents the slow collapse into a
cycle that cannot be untangled later.

### The axiom gate

This is the single most valuable gate. `verify.sh` does a clean rebuild and runs
`Print Assumptions` on each headline theorem; the accepted set for a
mathcomp-analysis-based development is exactly:

```
boolp.functional_extensionality_dep
boolp.propositional_extensionality
boolp.constructive_indefinite_description
```

Anything else means something leaked. Note the rocq plugin's own `check_axioms.sh`
**omits `constructive_indefinite_description`** and so gives false alarms on
mathcomp-analysis proofs — use `rocq_verify` or your own list.

---

## 4. The proving workflow

**Statement-first.** Port a declaration as an `Admitted` skeleton whose docstring quotes
the paper statement and number, get the file compiling (`rocq_compile_file mode="vos"` is
a fast statements-only pass), and only then prove. This separates "did I state it right"
from "can I prove it", which are different problems and should fail separately.

**MCP-first, not `coqc`-first.** For scratch iteration on a single proof:

```
rocq_start file=<…>.v theorem=<lemma>
rocq_step_multi tactics=[...]      # try many candidates without advancing
rocq_check body=<…>                # commit tactics
```

Use `coqc`/`make` only for full rebuilds, axiom audits and final verification. Writing a
throwaway `/tmp/x.v` and compiling it re-pays the multi-second mathcomp import every
time; a warm `rocq_start` session pays it once.

**Search before writing tactics.** `rocq_query("Search …")` for exact matches,
`rocqet_search` for semantic ones (hosted, MathComp only, cold starts on free tier).
Check your own dictionary first. SSReflect idiom is where models are weakest — this is
measured, not folklore (Babel-Formal: 82.9% Lean→Rocq with interactive repair vs 16–33%
for vanilla→SSReflect), which is exactly why `mathcomp-skills` exists. Run
`/mathcomp-review --scope=changed` before each checkpoint.

**Commit constantly**, with only the files you touched. Never push, amend or reset from
inside a proving session.

### Hazards specific to the interactive tools

- Sessions read the `.v` file at `rocq_start`; later edits produce a `stale_warning`, and
  an external `make` is invisible to the session. Restart the session after editing.
- One pet process per MCP server *name*, so parallel subagents sharing the name share one
  pet. Use a named pool (`.claude/agents/rocq-prover-N.md`, each with its own inline
  `mcpServers` entry), one worktree per concurrent agent.
- mathcomp/analysis imports are RAM-heavy: set `ROCQ_MAX_PET_RSS_MB` per pool member and
  watch `rocq_diag`.
- `rocq_verify` checks that a proof proves the *given* statement — not that the statement
  is the paper's. Statement faithfulness is a human/review obligation, always.

---

## 5. Write the project's AGENTS.md

Once parts 1–4 are real, condense them into an `AGENTS.md` at the repo root covering:
the toolchain and the "never install ad hoc" rule; the build and gate commands; the
layering; the accepted axioms; the MCP-first workflow above; the style rules (below); and
where the spec lives. Keep it under ~100 lines — it is read every session.

**Style** (MathComp house style; `mathcomp-skills` is the reference): every file starts
`From <Namespace> Require Import prelude.`; `Section` + `Context`/`Variable`;
`Implicit Types`; 80-column lines; ssreflect tactics with `by`/`exact:` closers; names in
`mainSymbol_suffixes` form; `lra`/`nra`/`ring` for reals, `lia`/`nia` for `nat`.

**One `(** … *)` docstring per public declaration — and make it say what the declaration
means.** A docstring that is only a cross-reference (`(** Paper: Lemma 3.4. *)`) makes the
generated API documentation a useless translation table. Lead with the mathematical
content and put the provenance at the end:

```coq
(** A pointwise nonnegative function has nonnegative average.
    (Paper: Lemma 3.4.) *)
Lemma avg_ge0 T (f : T -> R) : (forall a, 0 <= f a) -> 0 <= avg f.
```

---

## 6. Blueprint and documentation site

A [rocqblueprint](https://github.com/reiniscirpons/rocqblueprint) blueprint mirrors the
paper statement-by-statement, with `\rocq{Namespace.file.lemma}` tags linking each claim
to its formal counterpart, and a dependency graph. It is the artifact that makes the
formalization legible to the paper's authors, and it doubles as the project plan. A
script that checks every cited declaration actually exists keeps it honest.

Traps, all hit in practice:

- **coqdoc shows proof scripts by default**, which is not what a reader wants. Build with
  the generated makefile's `gallinahtml` target for statements only. Put coqdoc flags in
  `CoqMakefile.local` (auto-included): `COQDOCEXTRAFLAGS = --index indexpage --toc-depth 2`
  moves the A–Z identifier index off `index.html`, so you can `cp html/toc.html
  html/index.html` and have `<site>/docs/` land on the table of contents.
- **Jekyll project sites need `baseurl`**, or every `relative_url` link points at the
  domain root and 404s: `baseurl: "/<repo>"` with `url: "https://<org>.github.io"` — not
  the repo path folded into `url`. Check the theme layout too: some patch around a
  missing baseurl by prepending the repo name, which then doubles.
- **MathJax**: `cdn.mathjax.org` was retired in 2017; use MathJax 3 from jsdelivr, and
  configure `inlineMath: [['$','$'], ['\\(','\\)']]` explicitly — `$…$` is *not* inline
  math by default.
- **Long Rocq identifiers overflow the PDF margins**: `Namespace.file.some_long_lemma` is
  one unbreakable word wider than a text line. Allow a break after each underscore
  (`\renewcommand{\_}{\textunderscore\allowbreak}`) plus `\sloppy`.
- Fetching a page to check rendered math does not work: page fetchers do not execute
  JavaScript, so MathJax never runs. Verify the *loader and config* are in the HTML, and
  check links with HTTP status codes.

---

## 7. CI

**One workflow, and build the development exactly once.** The natural split — a "gate"
workflow and a "docs" workflow — makes both compile the whole thing, doubling the cost
for nothing.

```
gates ────┐        static checks (layering, no Admitted): seconds, fails fast
blueprint ┴─> build ─> assemble ─> deploy
```

Keep the cheap static gates in their own dependency-free job so a layering mistake is
reported in seconds rather than after a 30-minute build.

### Caching, and the trap that makes it useless

Compiling mathcomp-analysis from source dominates CI (~17 min alone; ~30 min per job in
practice). `docker-coq-action` has **no caching of its own**, and no published image
ships mathcomp-analysis, so you must roll it:

- **opam switch** — have your install script tar the container's opam root into the
  workspace and let `actions/cache` key it on the hash of *that script* (keep the package
  list in the script, so the key and the packages cannot drift apart). This is the big
  win: 30m46s → 6m17s, measured.
- **build products** — worth much less, and easy to get *dangerously* wrong. Two rules:
  1. **Decide staleness by content, not timestamps.** Record a sha256 manifest of the
     sources; on restore, delete the `.vo` of every source whose hash moved. Timestamps
     cannot decide this: a commit authored before the previous run finished is *older*
     than the `.vo` that run cached, so `make` would skip a genuinely changed file and
     report a proof as checked without checking it.
  2. **coqdep makes every `.vo` depend on the `rocqworker` binary of the switch.** If you
     restore the switch from a tarball at build time, that binary is newer than every
     cached `.vo` and the whole cache buys nothing. After restoring the switch, inside the
     container, re-date the surviving build products. This is safe *because* rule 1
     already deleted anything whose source changed.

Note that a local test cannot catch trap 2: locally the Rocq binary predates your `.vo`,
so the cache appears to work. Verify caching behaviour by reading the CI log for what
actually recompiled, not by inferring it from a green check.

---

## 8. Definition of done

- `make` clean from scratch; zero `Admitted`/`admit`/`Axiom`/`Parameter`.
- `Print Assumptions` on every headline theorem lists only the three `boolp` axioms.
- Every statement traced to the paper by number.
- Blueprint builds, every `\rocq{}` citation resolves, dependency graph complete.
- `docs/DESIGN.md` dictionary and `docs/PLAN.md` status table current.

The first two are machine-checkable and you should wire them into CI on day one.

---

## Appendix A. The detail parts 1–8 compress

Surveyed 2026-09-10/12 by reading the repos, not their marketing. Counts and pins were
true then; treat them as a starting point to re-verify, not as gospel.

### A.1 The stack, layer by layer

```
coq-lsp 0.2.5+9.1            ships `pet` / `pet-server` — the petanque JSON-RPC server
  └─ pytanque v0.2.2         Python client (stdio / socket :8765 / HTTP)
      └─ rocq-mcp 0.3.1      13 MCP tools over `coqc` + `pet`; what the agent talks to
          ├─ rocq-skills     Claude Code plugin `rocq`: /rocq:prove|autoprove|checkpoint|…
          └─ mathcomp-skills MathComp style guide + /mathcomp-review + advisory edit hook
rocqet-search                hosted semantic search over MathComp (19,448 declarations)
rocq-mcp-evolve              independent OCaml MCP server (in-process Rocq); the skills do
                             not know its tools — optional, ignore unless benchmarking
```

pytanque's protocol types are generated from the coq-lsp `protocol.atd` of a given
version, so the pair is what is verified, not each half: **pytanque v0.2.2 ↔ coq-lsp
0.2.5+9.1 ↔ Rocq 9.1.1**. This is the real reason for the `< 9.2` pin in part 1.1.

The remaining org repos (rocq-ml-toolbox, Pile-of-rocq, LLM4Docq, deep-premise-research,
crrrocq, nlir, babel-formal) are ML-training and dataset pipelines: nothing to install for
interactive proving. Everything outside the org that gets cited in this space is Coq
8.x-pinned and unusable here — Rango (8.18, CoqPyt), Quarry (8.20 via the dead SerAPI),
CoqPilot (8.19, VS Code only), Tactician (≤ 8.19), Proverbot9001 — or is a benchmark
(ITPEval). coq-lsp/petanque is the only maintained programmatic interface to Rocq 9.x, and
rocq-mcp already wraps it. `coq-hammer.1.3.3+9.1` does exist and can be installed into the
switch, but its own TODO admits MathComp support is unsolved; expect `hammer` to do nothing
on bigop, canonical-structure or analysis goals, and `lia` after `zify` to be the real
hammer for arithmetic.

### A.2 The 13 rocq-mcp tools

| Tool | Purpose |
|---|---|
| `rocq_compile(source)` | batch `coqc` of a string; on in-proof error returns `state_id` + goals |
| `rocq_compile_file(file, keep_vo, mode, timing)` | whole-file `coqc`; multi-error walker; `mode="vos"` = statements-only pass |
| `rocq_verify(proof, problem_name, problem_statement)` | sandboxed check that a proof proves the *given* statement (Module sandbox, forbidden-command scan, `Print Assumptions` whitelist including the `boolp` trio) |
| `rocq_query(command, preamble\|file\|from_state)` | `Search` / `Check` / `Print` / `About` / `Locate` |
| `rocq_assumptions(name, file)` | `Print Assumptions` → axiom list |
| `rocq_start(file+theorem \| file+line+char \| preamble)` | open an interactive session → `state_id` + goals |
| `rocq_check(body, from_state)` | run tactics; returns `last_valid_state_id` on error, `proof_tactics` when done, `stale_warning` when the file moved under it |
| `rocq_step_multi(tactics[≤20], from_state)` | try many tactics *without* advancing |
| `rocq_toc(file)` / `rocq_notations(statement, preamble)` | outline / notation resolution |
| `rocq_diag()` / `rocq_health()` / `rocq_switch(name)` | pet health & RSS / switch & binaries seen / change switch |

Workspace resolution walks up from the file for `_RocqProject` / `_CoqProject` /
`dune-project` and parses `-Q`/`-R`/`-I` plus an allow-list of `-arg`s. Setting
`ROCQ_WORKSPACE` overrides that *and constrains every path*, so prefer auto-detection and
an in-repo gitignored `scratch/`. Other env knobs and their defaults: `ROCQ_PET_TIMEOUT=30`,
`ROCQ_QUERY_TIMEOUT_CAP=300`, `ROCQ_COQC_TIMEOUT=60`, `ROCQ_VERIFY_TIMEOUT=120`,
`ROCQ_MAX_PET_RSS_MB=min(50% RAM, 16384)`, `ROCQ_MAX_STATES=1000`, `ROCQ_DUNE_BUILD=1`.
Plugin sub-agents ignore `mcpServers` frontmatter, so a parallel pool has to be plain
`.claude/agents/*.md` files.

### A.3 The exemplars, and what each is worth stealing

| Repo | What it is | Size | Take from it |
|---|---|---|---|
| `digraph-theory` | tournaments on MathComp + coq-graph-theory | 34 files / 7.0k lines | the reference build/CI/opam/docs hygiene: coq-community Makefile, `meta.yml`, both `rocq-*.opam` and `coq-*.opam`, gating `build.yml` + non-blocking docs workflow, Python oracles under `scripts/` |
| `mathcomp-eulerian` | Stanley EC1 §1.3–1.6, Eulerian numbers, cd-index | 49 / 22.5k | blueprint + coqdoc + a "formal companion" PDF on Pages; Dockerfile baking Rocq + MC + coq-lsp + rocq-mcp + the vendored `rocq` plugin, so the agent setup travels with the repo |
| `icones-rocq` | Ehrhard–Geoffroy "Integration in Cones" on analysis | 77 / 75.5k | `tools/check_layers.py`, `verify.sh`, and `PLAN.md`/`AUDIT.md`/`docs/PAPER.md` as the documents that actually steer the agent |
| `mathcomp-qbs` | quasi-Borel spaces, probability monad | 13 / 9.0k | proof that this works: 414 proofs, 0 `Admitted`, agent-written with rocq-mcp; analysis house style (`R : realType`, file-header docs) |
| `small-prime-gap` | Maynard `M_105 > 4` by exact witness | 20 / 9.6k | exact `=` opam pins plus a comment recording the last validated full rebuild; `AUDITOR_CHECKLIST.md` and `SPEC_TO_PAPER.md` |
| `mathcomp-kummer` | Kummer's theorem | 4 / 613 | the minimal starter skeleton, and `blueprint/update_status.py` |
| `graph-theory-rocq` | ~1,700 open conjectures as axiom-free `Prop`s | 463 / 59.3k | `meta/` gate scripts, incl. the vacuity probe; `make gate` checking only *landed* milestones are Admitted-free |
| `Putnam2025-Rocq` | 12 Putnam 2025 problems, all solved | 24 / 7.7k | `verify.py` driving `rocq_verify` as the whole build system, when statements are the deliverable |

Two facts that surprise people: **no formalization repo in the org carries a
`CLAUDE.md`/`AGENTS.md`** (`.claude/` is gitignored) because the maintainers rely on the
user-level plugins — part 5 still says write one, since a repo that travels needs its
conventions in-tree. And the steering documents that do exist are plain `PLAN.md` /
`docs/DESIGN.md` / `NEXT_SESSION.md`, plus honesty artifacts written for humans
(`REPORT.md`, `AUDIT.md`, README sections on machine authorship and the axiom budget).

### A.4 Build, CI and ignore conventions actually observed

Every formalization repo uses `_CoqProject` + `coq_makefile`/`rocq makefile`; **none** uses
dune, nix or flakes. The `_CoqProject` always carries one `-R theories <Ns>`, warning flags
via `-arg -w -arg …`, and an explicit, commented, dependency-ordered file list — never a
glob. CI is `coq-community/docker-coq-action@v1` in one of two flavours: `opam_file:
<pkg>.opam` + `coq_version: '9.1.1'`, or `custom_image:
'mathcomp/mathcomp:2.5.0-rocq-prover-9.1'` + `install: opam install -y <extras>`; both need
the `sudo chown -R 1000:1000 .` before_script and the `sudo chown -R 1001:116 .` revert.
Docs workflows regenerate artifacts and fail on `git diff --exit-code`.

Standard `.gitignore`: `*.vo *.vok *.vos *.glob *.aux .*.aux Makefile.coq Makefile.coq.conf
.Makefile.coq.d .lia.cache .nia.cache rocq_mcp_cache_*.v .claude/ .venv/`.

### A.5 Gates beyond the axiom gate

Part 3's axiom gate is the one with the best return, but the org relies on six, and the
cheap ones are worth wiring in on day one:

1. **grep gate** — no `Admitted|admit.|Axiom|Parameter|Conjecture|Hypothesis` at top level
   outside comments. Strip nested comments before grepping or you will chase ghosts: most
   of the `Admitted` "hits" in the org's repos are the word inside a comment.
2. **`Print Assumptions`** on each headline theorem (part 3).
3. **Statement faithfulness** — extract the paper's statements verbatim at build time and
   render them next to the Rocq, document the definition closure, and keep a
   definition-by-definition audit file. This is the only defence against the failure mode
   `rocq_verify` cannot see.
4. **Independent oracle** — a Python brute-force/Monte-Carlo checker, pytest-ed in CI,
   agreeing with the Rocq definitions on small `n`. Catches a mis-stated definition that
   still typechecks.
5. **Vacuity probe** — try to prove each statement *and* its negation by automation; both
   must fail. Catches a statement that is trivially true as written.
6. **Layering** (part 3).
