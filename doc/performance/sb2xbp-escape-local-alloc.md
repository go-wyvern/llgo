# Package-local escape analysis: sb2xbp comparison

Date: 2026-11-04. Branch: `escape-local-alloc`, based directly on
`main` commit `792862ec0b7cce2082a1211aa3525024f8023c2d`.

This experiment extends the existing named-result optimization to ordinary
address-taken locals. It removes a heap allocation when every use of its address
can be proved to finish within the invocation. This is a conservative first step
toward general escape analysis.

## Implementation

`cl/local_alloc.go` follows field/index addresses, pointer conversions and phi
nodes. Reads, writes through the address, comparisons, and synchronous calls to
inspectable Go helpers in the same package can borrow the storage. Every matching
argument position must borrow it; using the same pointer in a second escaping
argument keeps the object on the heap.

Parameter summaries are cached per package. For recursive calls, the analysis
builds a dependency graph and propagates escapes to all dependent summaries
before caching any results. A recursive cycle is safe only if all its uses are
safe. Each allocation has a budget of 1,024 reference visits; exhausting the
budget leaves the allocation on the heap and publishes no incomplete summaries.
The shared Go SSA graph is unchanged.

Promotion is limited to allocations in the entry block, so separate loop
iterations keep distinct heap objects. Functions with a recover block remain
conservative. Returned/stored addresses, interface boxing, slice views, closures,
unknown/dynamic/native calls, `go`, `defer`, and integer pointer conversions are
not accepted by this analysis. The existing append/capture optimizations remain
separate. Zero-sized heap allocations retain the shared sentinel, oversized
locals retain the existing stack-size fallback, and explicit-root targets keep
pointer-containing storage on the scanned heap. Stack storage still uses the
existing zero initialization.

## Workload and measurement

- macOS 26.4.1, darwin/arm64, Go 1.27.1.
- sb2xbp: `24ba44e4583dd25f4fb330291ed06405d8b69f44`, including the user's
  existing `convert/_test/run.sh` and `xbpio/xbpio_test.go` edits.
- Original: `/Users/wuxinyi/go/src/github.com/goplus/sb2xbp`.
- Test copy: `/private/tmp/llgo-escape-validation/sb2xbp`. Go source files and
  `go.mod` were checked against the original. The original was not modified.
- Both builds include the same temporary `convert/noscan_metrics_test.go`
  `TestMain`, which reads allocation/collection counters around `m.Run()` and
  enables BDWGC timing. It does not force a collection.
- Measurements run serially; compiler tests, profiling and allocation probes do
  not run concurrently with timed measurements.
- Complete `./...` invocations use a new empty `GOCACHE` for every version and
  round, `LLGO_BUILD_CACHE=off`, `-p=1`, and `-count=1`.
- Prebuilt execution uses a separate process and `-test.count=1` each time. Each
  version gets one discarded warmup followed by seven interleaved samples.
- File-system and module-download caches remain warm. These are not measurements
  of a machine with all operating-system caches flushed.
- Wall time uses Python `perf_counter`; CPU time and peak RSS use `os.wait4`.

`GOFLAGS=' '` is deliberately a single space: the machine's stored GOFLAGS
contains an option unsupported by LLGo, and an empty environment value would
fall back to that stored setting.

## Full ./... tests, including compilation and linking

All ten tested packages passed in all six invocations. Three interleaved rounds
per version, with a fresh Go build cache and LLGo build caching disabled:

| Round | main wall time (s) | escape wall time (s) |
| --- | ---: | ---: |
| 1 | 80.275534 | 75.869033 |
| 2 | 75.971478 | 75.713224 |
| 3 | 76.756809 | 77.526331 |
| Median | 76.756809 | 75.869033 |

The measured median is **1.16% lower**, but the ranges overlap and the optimized
version is slower in round 3. These samples do not establish a stable end-to-end
speedup of that size. In particular, the 5.5% difference in the first pair must
not be reported as the optimization's effect.

| Metric, median of three | main | escape-local-alloc |
| --- | ---: | ---: |
| Sum of package test durations | 22.815 s | 22.230 s |
| Wall time minus package test durations | 53.635 s | 53.639 s |
| User CPU time | 96.940 s | 96.618 s |
| System CPU time | 10.170 s | 10.147 s |
| Whole-command peak RSS | 2,274,492,416 B | 2,285,846,528 B |

The remainder includes package loading, compilation, linking and process
overhead; it is not a separately instrumented compiler timer. Per-metric medians
need not add up to the median wall time. Most full-command time remains outside
test execution, which limits the total benefit of a runtime allocation change.

## Prebuilt TestAllProjects execution

All 367 project cases passed. Medians of seven fresh processes:

| Metric | main | escape-local-alloc | Change |
| --- | ---: | ---: | ---: |
| Wall time | 7.578595 s | 7.327607 s | -3.31% |
| BDWGC cumulative allocated bytes | 8,019,977,426 | 7,517,720,610 | -6.26% |
| Collections | 95 | 80 | -15.79% |
| BDWGC full-GC timer | 5,305 ms | 4,600 ms | -13.29% |
| Peak RSS | 482,689,024 B | 483,180,544 B | +0.10% |

The GC timer is `GC_get_full_gc_total_time`, not Go's `PauseTotalNs`, and must
not be added to wall time. Reduced cumulative allocation did not produce a
measurable reduction in peak RSS in this workload.

| Round | main wall time (s) | escape wall time (s) |
| --- | ---: | ---: |
| 1 | 7.564529 | 7.399158 |
| 2 | 7.577907 | 7.401720 |
| 3 | 7.537770 | 7.327607 |
| 4 | 7.578595 | 7.269963 |
| 5 | 7.610998 | 7.257721 |
| 6 | 7.604568 | 7.332266 |
| 7 | 7.627215 | 7.324335 |

Separate runs interposed `GC_malloc` and `GC_malloc_atomic` to verify that
allocations were actually eliminated. Probe timings are excluded above.

| Native allocation counter | main | escape-local-alloc | Change |
| --- | ---: | ---: | ---: |
| `GC_malloc` calls | 181,523,726 | 163,143,625 | -10.13% |
| Bytes requested from `GC_malloc` | 5,723,425,693 | 5,449,539,145 | -4.79% |
| `GC_malloc_atomic` calls | 0 | 0 | — |

These counters include startup and count these two BDWGC entry points only;
requested bytes differ from BDWGC's cumulative allocation accounting.

The controlled prebuilt runs show a repeatable runtime improvement and the
allocation probe confirms actual heap allocation removal. This first step still
leaves most allocation calls in place; it does not close the previously observed
gap to Go. More general slice/interface handling, non-entry allocations and
cross-package summaries need separate work and validation.

Raw measurements, expanded commands and instrumentation source are preserved in
[sb2xbp-escape-local-alloc.json](sb2xbp-escape-local-alloc.json). All percentages
in this report use `main` as the denominator.

## Commands

The main executable and clean source snapshot are preserved from
`792862ec0b7cce2082a1211aa3525024f8023c2d`. The optimized compiler was built with:

```sh
cd /Users/wuxinyi/go/src/github.com/goplus/llgo
GOFLAGS=' ' GOCACHE=/private/tmp/llgo-go-cache \
  go build -o /private/tmp/llgo-escape-validation/llgo-escape ./cmd/llgo
```

Build the two runtime comparison binaries:

```sh
cd /private/tmp/llgo-escape-validation/sb2xbp
GOFLAGS=' ' GOCACHE=/private/tmp/llgo-go-cache LLGO_BUILD_CACHE=off \
  LLGO_ROOT=/private/tmp/llgo-noscan-validation/main-root \
  /private/tmp/llgo-noscan-validation/llgo-main test -c \
  -o /private/tmp/llgo-escape-validation/sb2xbp-main.test ./convert
GOFLAGS=' ' GOCACHE=/private/tmp/llgo-go-cache LLGO_BUILD_CACHE=off \
  LLGO_ROOT=/Users/wuxinyi/go/src/github.com/goplus/llgo \
  /private/tmp/llgo-escape-validation/llgo-escape test -c \
  -o /private/tmp/llgo-escape-validation/sb2xbp-escape.test ./convert
```

Each prebuilt run uses the following command, substituting `main` or `escape`:

```sh
cd /private/tmp/llgo-escape-validation/sb2xbp/convert
LLGO_ROOT=/Users/wuxinyi/.llgo/current \
  /private/tmp/llgo-escape-validation/sb2xbp-main.test \
  -test.run='^TestAllProjects$' -test.count=1 -test.timeout=10m
```

Full tests include compilation, linking and every tested package:

```sh
cd /private/tmp/llgo-escape-validation/sb2xbp
main_cache=$(mktemp -d /private/tmp/llgo-escape-validation/cache-main-XXXXXX)
GOFLAGS=' ' GOCACHE="$main_cache" LLGO_BUILD_CACHE=off \
  LLGO_ROOT=/private/tmp/llgo-noscan-validation/main-root \
  /private/tmp/llgo-noscan-validation/llgo-main test -p=1 -count=1 ./...

escape_cache=$(mktemp -d /private/tmp/llgo-escape-validation/cache-escape-XXXXXX)
GOFLAGS=' ' GOCACHE="$escape_cache" LLGO_BUILD_CACHE=off \
  LLGO_ROOT=/Users/wuxinyi/go/src/github.com/goplus/llgo \
  /private/tmp/llgo-escape-validation/llgo-escape test -p=1 -count=1 ./...
```

The measurement driver is `/private/tmp/llgo-escape-validation/compare.py`:

```sh
python3 /private/tmp/llgo-escape-validation/compare.py runtime
python3 /private/tmp/llgo-escape-validation/compare.py full
```

It records the fully expanded command, log path, exit code and process statistics
for every sample in `runtime.json` and `full.json` in that directory.

## Compiler validation

Passed compiler tests cover ordinary scalar/struct/array locals, pointer fields,
named results, self/mutual recursion, propagation through escaping cycles,
parameter-cache order, budget exhaustion, multiple argument positions,
unsupported operations, debug SSA, and darwin/arm64, linux/amd64, windows/386 and
wasip1/wasm IR. Runtime regression checks exercise GC while a borrowed local is
live, escaping recursive helpers, deferred and returned closures, interface
boxing, and distinct loop allocations.

Commands from the repository root:

```sh
GOFLAGS=' ' GOCACHE=/private/tmp/llgo-go-cache go test ./ssa -count=1
GOFLAGS=' ' GOCACHE=/private/tmp/llgo-go-cache \
  LLGO_ROOT=/Users/wuxinyi/go/src/github.com/goplus/llgo \
  go test ./cl -run '^(TestCompile.*Locals|TestBorrowed.*|TestCompile.*NamedResults|TestResultEscapeAnalysis.*|TestNamedResultExplicitRoots)$' -count=1
```

Compiler integration tests used a Go overlay that redirects the existing
`internal/build.cacheRootFunc` test hook to the initially empty
`/private/tmp/llgo-escape-validation/codegen-cache`. This avoids importing stale
archives from other compiler branches in the user's global LLGo cache. The
overlay only changes the cache location; it is not part of the implementation.

```sh
GOFLAGS=' ' GOCACHE=/private/tmp/llgo-go-cache \
  LLGO_ROOT=/Users/wuxinyi/go/src/github.com/goplus/llgo \
  go test -overlay /private/tmp/llgo-escape-validation/overlay.json ./cl \
  -skip '^TestRunAndTest' -count=1 -timeout=15m
GOFLAGS=' ' GOCACHE=/private/tmp/llgo-go-cache \
  LLGO_ROOT=/Users/wuxinyi/go/src/github.com/goplus/llgo \
  go test -overlay /private/tmp/llgo-escape-validation/overlay.json ./cl \
  -run '^(TestRunAndTestFromTestgo|TestRunAndTestFromTestrt|TestRunAndTestRuntimeCodegen)$' \
  -count=1 -timeout=20m
```

The integration run passed all execution checks. Two remaining IR expectations
(`index` and `typalias`) were then corrected and their subtests passed on rerun.
