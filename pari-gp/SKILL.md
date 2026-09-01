---
name: pari-gp
description: Use before writing, editing, or debugging a PARI/GP (.gp) script, or before running one with gp. Covers recurring gp-specific footguns -- script-file parsing rules for `{ }` blocks, setting `parisize` inside a function (which kills the run silently), `subst` vs `substvec`, `ffgen` semantics, `my()`/closures, variable priority in `nffactor`, a `vecsum` type trap, `~` being transpose and not bitwise NOT, floating-point leaking into exact computations, and syntax errors that make gp skip the file and still exit 0 (lint with `gp2c` first) -- and the Claude Code convention of creating .gp files with the Write tool rather than shell heredoc.
---

# PARI/GP scripting: recurring pitfalls

These are traps that have each cost real debugging time, more than once, across
independent sessions. Read this before writing or editing a `.gp` file, and
re-check it when a script's output looks wrong but produces no error.

## Creating and editing files

- **Create `.gp` files with the Write/Edit tool, not `printf`/heredoc via a
  shell.** Heredoc has silently corrupted backslash escapes (e.g. `\2`
  vanishing entirely). If a shell redirect is unavoidable, diff the written
  file against the intended content before running it.

## Script structure

- **Wrap all processing inside functions; nothing free-floating at the top
  level.** Give each piece of functionality its own function, write a
  driver function `run()` that calls them in the right order, and put a
  single `run();` as the file's last line. Every variable is
  `my()`-declared inside some function — no bare top-level assignments
  living outside a function body.
  **Don't name the driver `main()`**: if this script is ever run through
  `gp2c` (the GP-to-C transpiler), `main` is a reserved C symbol and will
  fail to compile/link. `run()` was verified clash-free — compiled, linked,
  loaded, and called successfully via `gp2c` end to end.

## Stack size

- **Set `parisize` at top level, never inside a function.**
  `default(parisize, ...)` reallocates the PARI stack, and the reallocation
  *aborts the computation in progress*. At top level that costs only the one
  statement; inside the driver function it kills the entire run — the
  stack-size warning prints, nothing else does, and the exit code is `0`, so
  it looks exactly like a script that silently produced no output. Verified
  twice, in two different scripts, in one session.
  **This interacts badly with the house style above**: once every statement
  lives inside `run()`, `default(parisize, ...)` is the one line that must
  stay outside it.

## Block syntax in script files (not the interactive REPL)

Interactive `gp` and script files parse multi-line constructs differently.
In a script **file**, all of the following are real, repeat-offender traps:

- **House style: always write `funcname(args) = { ...; return(retval); }`.**
  Put the `{` immediately after `=`, end the body with an explicit
  `return(...)`, and close with `};`. GP also accepts wrapping the entire
  `name(args) = ...` statement in an outer `{ }` (brace *before* the name)
  and relying on the last bare expression as an implicit return — both
  parse correctly — but this project sticks to the one form above so
  there is nothing to double-check later.
- **Any multi-line construct needs explicit `{ }`.** `for(...)`, `if(...)`,
  a multi-line function body, or a multi-line vector/statement list — without
  `{ }` wrapping the whole thing, gp silently takes only the *first physical
  line* as the body and keeps going. There is no error message. Symptom: a
  function that should loop or branch just returns after one step, or a
  helper quietly returns a bare `t_INT` instead of doing anything.
  **Wrap every multi-line block in `{ }`, no exceptions.**
- **`{ }` blocks do not nest.** `{ ... { ... } ... }` is not supported.
  If a helper needs its own block, pull it out into a separate top-level
  function instead of nesting.
- **The mirror-image trap: a one-line `name(args) = ...` swallows the rest of
  the physical line.** Don't put another statement, or even a trailing
  comment, after such a definition on the same line — give each
  `name(args) = ...` its own line. (This is the opposite failure mode from
  the missing-`{ }` trap above: one-line defs eat too much, multi-line defs
  without `{ }` eat too little.)

## Semantic traps

- **`subst` chains apply sequentially, not simultaneously.** Chaining
  `subst(subst(p,x,a),y,b)` substitutes into the *already-substituted*
  expression, which is usually not what "substitute x->a and y->b" means.
  Use `substvec(pol, [vars], [vals])` for a genuine simultaneous
  substitution.
- **`my(f(x,y) = ...)` does not create a callable local function.** It
  leaves an unevaluated `t_CLOSURE` sitting where a value is expected, and
  fails downstream with something like `incorrect type in ... (t_CLOSURE)`
  — far from the actual mistake. Define the function at top level, or use
  the arrow form inside `my()`: `my(f = (x,y) -> ...)`.
  Note this is *only* a `my()`-declaration problem: `f = (x,y) -> ...` on
  its own is a proper GP closure and can be called normally.
- **`ffgen(p^k)` is a root of the defining polynomial, not a multiplicative
  generator of the finite field's unit group.** For a prime field (`k=1`)
  it typically returns `0`. Enumerating $\mathbb F_q$ (or $\mathbb F_q^\times$)
  as `{ g^i : i }` using `g = ffgen(q)` visits almost none of the field —
  silently, with no error, since the powers are still valid field elements.
  If you need a multiplicative generator, compute one explicitly (e.g. via
  `znprimroot` for the prime case, or search among nonzero elements checking
  the order), or enumerate by digits over a fixed basis instead of by powers
  of `ffgen`'s output.
- **`for` has no step argument.** `for(i = 6, 0, -1, ...)` is not a descending
  loop — the third argument is the body, so this is a syntax error at best and
  a silently wrong loop at worst. Use `forstep(i = 6, 0, -1, ...)`.
- **`~` is postfix transpose, and nothing else.** It is not bitwise NOT (the
  C habit) and it is not a prefix operator. Both mistakes fail far from where
  they read: `~(1<<k)` is a *syntax* error (`unexpected '(', expecting
  variable name`), and `0~` is a *runtime* error (`incorrect type in gtrans
  (t_INT)`) because it transposes an integer. For a zero column vector write
  `vectorv(n)`, not `0~`. To drop row and column `i` from a matrix, don't try
  to complement a bitmask — pass an index vector:
  `my(idx = select(j -> j != i, [1..n])); vecextract(M, idx, idx)`.
- **Variable priority is enforced, and the number field's variable must have
  the *lower* priority.** `nffactor`, `polrootsmod` and friends reject a
  polynomial whose variable does not outrank the field's: define the field by
  `y^3 - 2` and factor polynomials in `x`, not the other way round. Symptom:
  `incorrect priority in nffactor: variable t >= a`. Priority follows creation
  order, so a field defined with a variable you introduced earlier in the
  session will fail for no visible reason.

## Reading return values

- **Don't guess a compound return value's component order — look it up.**
  `?funcname` prints a one-line summary from the shell; `??funcname` prints
  the full manual entry with a worked example, non-interactively:
  `echo "??elltors" | gp -q`.
- **Concrete case that actually bit us**: `elltors(E)` returns `[t, v1, v2]`
  — `t` (component `[1]`) is the torsion group's **order**, `v1`
  (component `[2]`) is its **structure** as a product of cyclic groups.
  Reading `elltors(E)[1]` as "the structure" reported $\Z/4$ where the
  group was actually $(\Z/2)^2$ — both have order 4, which is why the bug
  wasn't obvious — and a later enumeration over the group silently ran
  over only half its elements.

## Type traps

- **`vecsum` rejects `t_VECSMALL`.** Wrap the argument in `Vec(...)` first
  if it might be a `t_VECSMALL` (e.g. output of `vecextract`,
  `factor(...)[,2]`, or similar).

## Exactness traps

- **Fractional/real exponentiation silently goes through floating point,**
  even in the middle of an otherwise fully exact integer computation.
  `x^(1/n)` on an integer `x` does *not* stay in the exact integer domain —
  it becomes a real number, and any later "is this equal to an integer"
  check inherits float error. If a computation needs to be exact end-to-end
  (i.e. anything claimed at `[T(CAS)]` grade or stronger), use integer-only
  operations such as `sqrtnint`, `ispower`, or explicit rational arithmetic
  instead, and audit the chain for any stray `^(1/...)` or other implicit
  float step.

## Invocation

- Scripts are normally run as `gp -q script.gp` (quiet mode, suppresses the
  startup banner).
- **`gp -q script.gp` can hang forever with no error if stdin isn't
  closed.** After the script finishes, gp drops to an interactive prompt
  and waits for more input. For any non-interactive or backgrounded
  invocation, close stdin explicitly: `gp -q script.gp < /dev/null`.
- **A syntax error anywhere in the file makes gp skip the whole file and
  still exit 0.** It prints `*** syntax error ...` followed by
  `... skipping file 'x.gp'`; with the house style above (one `run();` on the
  last line) a `*** not a function in function call` follows, because the
  driver was never defined. Nothing runs, and **the exit status is 0**. So the
  exit code alone cannot tell "every gate passed" from "the file never ran" —
  a runner must *also* require the gate line in the output. (This is the same
  shape as the `parisize` trap above, but general: any syntax error does it.)
  gp does not print a line number either.
- **Lint with `gp2c` first — it exits 1 and gives the line number.** Measured
  on the same broken file: `gp -q syn.gp` printed the error and exited `0`;
  `gp2c syn.gp` printed `syn.gp:2: syntax error, unexpected '~'` and exited
  `1`. Run it as a pre-flight check, and note that it only reads the file:
  ```sh
  gp2c script.gp > /dev/null || echo "syntax error"   # lint, exits 1 on error
  gp -q script.gp < /dev/null                          # then run
  ```
  It is conservative: on five working scripts from a real project it reported
  no errors. Two kinds of *warning* are normal and harmless: `function
  prototype is unknown <f>` for built-ins gp2c has not been taught (e.g.
  `ellrank`), and `meta commands not implemented \q`. A type error such as
  `0~` is *not* caught — that one is only found at runtime.
- **Verify the failure path once.** Run a deliberately broken copy (flip one
  `check` to something false) and confirm the script exits non-zero. It is
  cheap, and it catches a mis-wired `quit(1)` or a gate that reports RED while
  still exiting 0.

## Monitoring long-running jobs

- **Never pipe a long-running job directly into `tail`.** `cmd | tail -n
  20` shows *nothing* until `cmd` closes its stdout — i.e. until it
  finishes — because `tail` must see the end of the stream to know what
  "the last N lines" are. Verified: a script piped through `tail`
  produced zero bytes of output for its entire ~26s run, then dumped
  everything at once at exit — indistinguishable from a hung process.
  **Redirect to a file instead** (`gp -q script.gp > run.log 2>&1 </
  dev/null`) and watch the file (`tail -f run.log`, or just re-read it).
  This works: gp flushes each `print`/`printf` immediately even when
  stdout is not a terminal (verified — output appeared in the log file
  within ~2s of being printed, no block-buffering on gp's side).
- **This is a `tail` problem, not a gp buffering problem** — gp's own
  output is not block-buffered when redirected, so the fix above is
  sufficient for gp. (SageMath has a related but different, genuinely
  Python-specific issue — see the separate `sagemath` skill.)

## gp2c beyond linting

`gp2c` is a GP-to-C translator, not only a syntax checker. It compiles a `.gp`
script to C, builds it as a shared library and loads the compiled functions
into gp — `gp2c-run script.gp` launches such a session, and `install()` binds a
function from an already-built library. That is the standard way to speed up a
hot GP loop without rewriting it in C.

Two conventions in this file exist because of it: the driver is named `run()`
and not `main()` (a reserved C symbol), and the lint step above. If you compile
for real rather than only linting, the subset restrictions that the lint
reports as mere warnings — unknown prototypes for built-ins gp2c has not been
taught, meta commands — become real limits.
