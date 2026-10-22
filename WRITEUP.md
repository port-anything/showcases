# Six coreutils, three jq implementations, one matrix: 12 new divergences, 8 already fixed upstream

We ran [Port Anything](https://github.com/karolkulesza/port-anything) against five reimplementations of programs most of us use every day: **uutils, busybox and toybox** against GNU coreutils, and **jaq and gojq** against jq. Each candidate got the same generated cases as the reference, run side by side in a sandbox. Here is how close each one gets to the reference.

| GNU coreutils 9.12 → | uutils 0.12.0 | uutils `main` | busybox 1.38.0 | toybox 0.8.15 |
|---|---|---|---|---|
| `od` | 90.5% | 90.5% | 54.3% | 9.4% |
| `nl` | 81.3% | 81.3% | 48.5% | 14.3% |
| `comm` | 97.4% | 99.6% | 11.3% | 11.3% |
| `fmt` | 88.4% | 92.6% | — | 11.3% |
| `join` | 97.8% | 98.6% | — | — |
| `numfmt` | 99.9% | 99.9% | — | — |

| jq 1.8.2 → | jaq 3.1.1 | gojq 0.12.19 |
|---|---|---|
| parity | 72.1% | 81.2% |

Parity is the mean match rate per feature (a flag, or an input form), so a rarely used flag weighs as much as the default path. "—" means the utility isn't in that implementation's default build. **Read the busybox and toybox columns as GNU compatibility, not quality**: both are deliberately small, and options they don't implement count as divergences here. The jq numbers are raw: nothing is waived, and most of the gap is behavior that jaq and gojq document as intentional.

The part we care about more than the percentages: **12 bugs nobody had reported**, each reduced to a one-line repro and checked against the project's tracker. Nine come from the published runs. The first three come from an earlier prototype run of the same approach (about 900 cases on `od`, `nl` and `numfmt`), were filed right away, and **all three are already fixed upstream**; the published runs don't re-find them, so a rerun of the showcases shows the other nine. We filed those too, and the maintainers were quick: uutils fixed all three of its new ones within a day, jaq fixed its crash and documented the other two differences as intentional, and gojq changed `isnan` to match jq. Of the 12, **8 are fixed upstream** now, 2 are documented by the project, the toybox one is open, and the busybox one we haven't reported yet (busybox takes reports on its mailing list). Everything is in the [showcases repo](https://github.com/port-anything/showcases): every case, both runs, the HTML reports and the exact versions.

## Why these utilities

uutils already tracks its pass rate on the GNU test suite, its CI fuzzes several utilities against GNU, and two days before our first run someone published [~180 differences found by an AI differential fuzzer](https://github.com/uutils/coreutils/issues/15254) across all of coreutils. Re-running `sort` or `wc` would mostly re-find their work. So we picked utilities **without a uutils fuzz target**: `od`, `nl`, `comm`, `fmt`, `join` and `numfmt`. The first four ship in at least three of the implementations; `join` and `numfmt` only in GNU and uutils, so those two rows are a pairwise comparison.

And we added busybox and toybox, which none of the existing differential work covers. busybox is the userland of Alpine and countless embedded systems; toybox is Android's command line.

For jq, both [jaq](https://github.com/01mf02/jaq) and [gojq](https://github.com/itchyny/gojq) list their intentional differences from jq, which makes them a good test of whether a tool can tell "different on purpose" from "different by accident".

## How it works

For each utility, Port Anything reads the reference's `--help`, generates cases from it: flags with good and bad values, alone and combined, over small input files and stdin (UTF-8, tabs, a missing final newline, empty input, odd file names), plus byte mutants of those inputs. 1000 cases per utility (fewer after de-duplication), with a fixed seed, and each case runs on every implementation. For jq, the seeds are the 550 cases in jq's own `tests/jq.test`, plus 2920 mutants: filters and inputs with tokens replaced or inserted, or bracket-balanced pieces spliced in from other filters, because flipping random bytes in a jq filter mostly gives parse errors.

Then it does the work that turns a pile of diffs into something a maintainer can use:

- **Compare** exit status, stdout and stderr per case (jq: exit status and stdout only, since the three never share error wording; coreutils stderr is compared loosely, so an error message doesn't have to match word for word).
- **Cluster** divergences by what differs and how (exit 0 vs 1 with empty stdout, whitespace-only stdout difference, …), so dozens of diverging cases with one cause show up as one cluster.
- **Minimize** a representative case, then a person reproduces it by hand outside the tool, under `LC_ALL=C`.
- **De-duplicate** against the project's issues, PRs and failing-GNU-test tracker, and against the flagdiff report. Only what isn't there counts as new.

## The findings

| Project | Finding | Status |
|---|---|---|
| uutils | `numfmt` rejects scientific notation in fields it was told to leave alone: `echo "1000 1e3" \| numfmt --to=si` | fixed ([#15312](https://github.com/uutils/coreutils/issues/15312)) |
| uutils | `od -B` (the traditional alias for `-t o2`) rejected | fixed ([#15313](https://github.com/uutils/coreutils/issues/15313)) |
| uutils | `od -S` skips the "cannot skip past end of combined input" check | fixed ([#15314](https://github.com/uutils/coreutils/issues/15314)) |
| uutils | `numfmt` keeps a tab between fields; GNU writes a space: `printf '1\t2\n' \| numfmt` | fixed ([#15552](https://github.com/uutils/coreutils/issues/15552)) |
| uutils | `fmt -p PREFIX` adds a newline to an unprefixed last line that has none: `printf x \| fmt -p '#'` | fixed ([#15553](https://github.com/uutils/coreutils/issues/15553)) |
| uutils | `od -w1K` accepted (GNU rejects the suffix); `nl -l 18446744073709551616` rejected (GNU accepts) | fixed ([#15551](https://github.com/uutils/coreutils/issues/15551)) |
| busybox | `od` with output types of different sizes prints the last block's zero padding as input: `printf abc \| busybox od -c -x` shows `a b c \0` | not reported yet |
| toybox | `fmt` writes `\e[7m\e[27m` into its output for every word with a control byte or invalid UTF-8: `printf 'a\001b\n' \| toybox fmt` | open ([#615](https://github.com/landley/toybox/issues/615)) |
| jaq | panics (exit 101) instead of a syntax error when a filter ends inside a comment with a bracket open: `jaq -n '(#'` | fixed ([#476](https://github.com/01mf02/jaq/issues/476)) |
| jaq | stack-overflow abort (exit 134) on values nested a few thousand levels deep, where jq and gojq print them or stop with an error at 10000 | documented: jaq treats deep nesting as out of scope ([#477](https://github.com/01mf02/jaq/issues/477)) |
| jaq | `from_entries` reads only `key`/`value`; `name`, `Name`, `Key` and `Value` silently become a `null` key | documented as an incompatibility ([#478](https://github.com/01mf02/jaq/issues/478)) |
| gojq | `isnan` errors on non-numbers, where jq returns `false` | changed to match jq ([#327](https://github.com/itchyny/gojq/issues/327)); gojq had chosen this on purpose |

The first three were found by the prototype run; the other nine by the published runs.

A few of these say something about *how* to find bugs, not just where:

**Several depend on the input content, not the flags.** The `numfmt` tab, the toybox escape sequences and the busybox padding bytes only appear with particular bytes in the input (a tab between fields, a control byte inside a word, a file length that isn't a multiple of the widest output type). A fuzzer that varies flags over a fixed set of inputs is unlikely to hit these; varied and byte-mutated inputs did.

**The jaq panic came from mutation alone.** None of jq's 550 test cases ends inside a comment. A mutant that replaced one `.` with `#` in `del(.foo[0], .bar[0], …)` did, and minimizing it by hand gave `(#`.

**The toybox bug isn't about `fmt`.** `fmt` measures each word with `utf8len()`, and the reverse-video escapes come from a helper that writes even when it was only asked to measure. Other callers may be affected too.

### What we found but didn't count

Most divergences were already known, and we didn't report them again:

- `od -w FILE` reads the next word as the width (uutils [#10725](https://github.com/uutils/coreutils/issues/10725)); `od -f` float formatting ([#13608](https://github.com/uutils/coreutils/issues/13608)); `nl -v -1` rejected ([PR #12459](https://github.com/uutils/coreutils/pull/12459)); `fmt -g` line breaking ([#5162](https://github.com/uutils/coreutils/issues/5162)).
- `fmt` treating `\r`, `\v`, `\f` as blanks, `join` adding an empty field after a trailing blank, `od` dropping the final offset line for a missing file, `numfmt --from=a`: all in the flagdiff report ([#15254](https://github.com/uutils/coreutils/issues/15254)), which found them first.
- gojq's `abs` on non-numbers: already fixed on gojq's `main`, not released yet.

And some aren't bugs at all: `--help` and `--version` text, error-message wording, busybox and toybox `comm` not checking sort order (POSIX doesn't require it), options busybox and toybox don't implement, and for jq everything jaq and gojq document (sorted object keys, number precision, builtins jaq doesn't have, like `$__loc__` or `INDEX`, error text inside `try … catch`, NaN handling, depth limits).

## Keeping false positives out

A matrix like this is only worth publishing if a divergence means a real difference. What we did about it:

- **Pin the reference.** Our prototype run used the distro's GNU coreutils (Ubuntu 25.10 ships 9.5). Three of its "divergences" were GNU 9.5 behavior that later GNU releases changed (`od -w0` aborting, for one). The published runs use GNU 9.12, built from source.
- **Check the architecture.** A fourth one was the CPU: on arm64, GNU `numfmt` prints a 20-digit number thanks to a 128-bit `long double`, while on x86_64 it fails just like uutils. We caught it by re-running the repro under x86_64. The published runs and the nine new repros haven't had that x86_64 check yet (see Caveats).
- **Run the reference more than once.** Every case runs on the reference three times; cases whose output varies are left out instead of being blamed on the candidate. A candidate whose result changes on a re-run is marked flaky.
- **Same program name, same environment.** Every side runs under the reference's name (multicall binaries like `busybox` get `argv[0]` pinned), in a bubblewrap sandbox with no network, `LANG=C.UTF-8`, `TZ=UTC` and a fixed `SOURCE_DATE_EPOCH`.
- **Separate self-description from behavior.** `--help` and `--version` are reported on their own and don't count toward parity.
- **Reproduce by hand** before calling anything a bug.

## Caveats

- **Linux aarch64 only** (Ubuntu 25.10 container). Not repeated on x86_64 yet.
- **Coverage gaps.** The generator caps how many error-path cases it makes, and in these runs that left `join -a/-j/-o/-v/-1/-2` and `numfmt --to/--field/--format` without cases. `numfmt`'s 99.9% rests on 3 features with enough cases; the rest are listed as under-sampled in the report. `csplit` and `pr` were left out (`csplit` needs pattern-aware operands, `pr` prints a date).
- **Unimplemented options count as divergences.** That's most of the busybox and toybox gap. A future version will report them as untested instead.
- **jq: nothing waived.** Most of jaq's and gojq's divergences are documented, intentional differences. Expressing those as waivers would raise both scores; we'd rather show the raw numbers and the list.
- Parity is evidence from the cases that ran, never a proof of equivalence.

## Reproduce it

Each directory in the [showcases repo](https://github.com/port-anything/showcases) is an ordinary Port Anything project: `port-anything.yaml`, the committed cases, the CLI output of the baseline and the run, and the report. Inside a Linux container with the implementations at the paths in `port-anything.yaml`:

```sh
port-anything baseline && port-anything run && port-anything report
```

`versions.txt` in each directory lists the exact versions, the uutils commit, the distro and the kernel.

## Thanks

To the uutils, busybox, toybox, jaq and gojq maintainers, who build these for free; the uutils, toybox, jaq and gojq maintainers got the reports before this post, and most were fixed or documented within a day; and to the author of flagdiff, whose report saved us from re-filing several known issues.
