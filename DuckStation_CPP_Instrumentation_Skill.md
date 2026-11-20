# DuckStation C++ Source Instrumentation Skill

## Complete Execution Maps, Three Diagnostic Levels, Optional Forensic Tracing, and Build-Safe Editing

### Purpose

Use this skill when instrumenting a DuckStation `.cpp` file to build a
reliable map of how that subsystem executes.

The objective is to answer four questions:

1.  What are all important execution paths in this subsystem?
2.  How does data move through those paths, from entry to completion?
3.  Which state changes, decisions, timing events, and transfers explain
    the behaviour?
4.  What remains unobserved because it is outside this file, bypasses
    its handlers, or was not exercised by the test?

A **complete important execution map** is not the same as a log of every
repeated operation. Level 1 should map all important paths while
suppressing noise. An optional Level 4 forensic mode can record every
observable operation through the instrumented paths.

Logging must remain observational. It must not change emulated register
values, timing, FIFO order, memory contents, return values, control
flow, or subsystem behaviour.

This skill is based on the DuckStation GPU instrumentation approach and
can be adapted to BUS, CPU, DMA, GPU, CD-ROM, MDEC, SPU, timers,
controllers, memory, and other subsystems.

------------------------------------------------------------------------

## 1. Definitions: what does "full execution map" mean?

Always distinguish these three kinds of completeness:

### 1.1 Source instrumentation coverage

Which functions, branches, entry points, exits, state transitions, and
transfer paths contain instrumentation?

This is established by reading the complete source and relevant headers,
then auditing the edited source.

### 1.2 Runtime execution coverage

Which instrumented paths actually executed during a particular test?

A log proves only the paths exercised by that run. A handler can be
instrumented but remain untested if the workload never reaches it.

### 1.3 Observability boundary

Which operations can this source file actually observe?

For example, BUS instrumentation can observe accesses routed through the
BUS implementation. It cannot automatically show every internal
operation performed later inside DMA, GPU, SPU, CD-ROM, or another
subsystem. Those internal paths require instrumentation in their own
source files.

Never claim that one `.cpp` file captures the complete execution of an
entire multi-file subsystem unless the source architecture genuinely
routes all relevant operations through that file.

Use precise conclusions such as:

-   Complete important map of the instrumented BUS paths.
-   Partial runtime coverage: GPU GPUREAD was exercised, but other paths
    were not.
-   Internal DMA execution is outside this file's observation boundary.
-   Exhaustive for accesses routed through these handlers, but not for
    internal device activity.

------------------------------------------------------------------------

## 2. Non-negotiable rules

Before editing:

-   Read the complete target `.cpp`, not just the function named in the
    request.
-   Read the relevant headers, types, enums, register definitions, and
    call sites.
-   Inspect the original source and preserve it as the behavioural
    reference.
-   Inspect the supplied final instrumentation example and follow its
    quality, naming, structure, and level discipline where appropriate.
-   Build a subsystem map before inserting logs.
-   Make instrumentation-only changes. Do not rewrite algorithms to make
    logging easier.
-   Do not casually change `const`, `volatile`, `constexpr`,
    `ALWAYS_INLINE`, `RESTRICT`, calling conventions, templates, data
    types, or object layout.
-   Do not add logs to every function merely to increase line coverage.
-   Do not add extra reads, writes, FIFO operations, timing
    synchronization, or callbacks to obtain values for logging.
-   Never evaluate a side-effecting expression twice.
-   Do not modify shared or third-party headers to fix a formatter error
    caused by a new log statement.
-   Do not claim a build passed unless the exact target was built
    successfully.
-   Do not claim runtime validation unless the edited binary was
    actually run with a representative workload.

The purpose is to make important execution understandable, not to
produce the largest possible log.

------------------------------------------------------------------------

## 3. Logging levels

Use one compile-time trace-level setting local to the `.cpp` file
initially. Keep the setting easy to change and use the target file's
existing log channel and conventions.

Example for the standard three-level mode:

``` cpp
namespace
{
// 0 = OFF
// 1 = COMPLETE IMPORTANT EXECUTION MAP
// 2 = DETAILED OPERATION TRACE
// 3 = DEEP INTERNAL DIAGNOSTICS
static constexpr u32 TRACE_LEVEL = 1;
static_assert(TRACE_LEVEL <= 3, "TRACE_LEVEL must be between 0 and 3");

#define TRACE_LOG(level, ...)                         \
  do                                                  \
  {                                                   \
    if constexpr (TRACE_LEVEL >= (level))             \
      DEBUG_LOG(__VA_ARGS__);                         \
  } while (false)
} // namespace
```

Do not confuse these custom levels with DuckStation's normal logger
severity categories such as Debug, Trace, Info, or Error. The custom
level controls which instrumentation statements are compiled into the
trace.

### Level 1 --- Complete important execution map

**Question answered:** What important operations happened, in what
order, and through which paths?

Level 1 must cover every important architectural path identified during
source inspection. It is not enough to log only the most common path.

Include, where applicable:

-   Initialization, shutdown, reset, and state restoration.
-   Public entry points and CPU-facing interfaces.
-   Register/MMIO read and write categories when they represent
    important device interactions.
-   Command or request acceptance, rejection, queuing, decoding,
    dispatch, and completion.
-   DMA or other producer/consumer hand-off boundaries.
-   Major state transitions.
-   Transfer start, progress summary, completion, abort, and error
    paths.
-   Timing events that materially change execution.
-   Backend submission and completion boundaries.
-   Important fallback, invalid, unsupported, and exceptional branches.
-   Cache, memory, interrupt, and mapping changes that alter later
    execution.

Level 1 should allow a reader to reconstruct the subsystem's important
flow without seeing every helper call or repeated poll.

For repetitive activity, use aggregation or bounded samples rather than
removing the entire path from the map. Preserve at least the operation
category, address or target, direction, count, and meaningful start/end
boundary where those concepts apply.

### Level 2 --- Detailed operation trace

**Question answered:** How did each important operation process its
input and change state?

Add meaningful details to Level 1 boundaries:

-   Operation subtype, opcode, register, address, width, and direction.
-   Packet or transfer length, expected count, and actual count.
-   Decoded fields and relevant modes.
-   Source and destination information.
-   FIFO or queue occupancy when it explains behaviour.
-   DMA channel, direction, address, increment, synchronization mode,
    and word count.
-   State before and after important transitions.
-   Dispatch target and result.
-   Transfer parameters and completion reason.
-   Backend command type and important parameters.

Do not dump every payload element at Level 2. Explain the operation, not
every byte.

### Level 3 --- Deep internal diagnostics

**Question answered:** Which exact values or internal decisions explain
a difficult result?

Use selectively for:

-   Raw register values and decoded bitfields.
-   Selected memory, FIFO, packet, or transfer samples.
-   Internal state snapshots.
-   Rare branches and fallback decisions.
-   Detailed timing and counter values.
-   Return values where they provide useful evidence.
-   First-five/last-five data samples.
-   Before/after values for state changes.
-   Address translation, masking, alignment, and mapping details.

Level 3 should expose useful evidence without turning every hot path
into an unbounded stream.

### Optional Level 4 --- Full forensic transaction trace

Use Level 4 only when explicitly requested or when Level 1--3 cannot
answer a specific question.

**Question answered:** What exact observable operations passed through
the instrumented code, including repeated operations?

Where technically possible, Level 4 should:

-   Record every observable read/write/transaction routed through the
    instrumented paths.
-   Disable change-only suppression and first/last sampling for the
    operations declared exhaustive.
-   Preserve execution order.
-   Include address, width, direction, value/result, and relevant caller
    or transaction context when already available.
-   Avoid adding extra hardware reads or synchronization.
-   Clearly identify paths that bypass the instrumented handlers.
-   State any unavoidable exclusions.

Level 4 is not automatically a complete hardware execution trace. It is
exhaustive only within the explicitly declared observation boundary.
Internal DMA/GPU/SPU/CD-ROM activity, fast-memory paths, direct
pointers, cached/recompiler paths, or backend work may bypass a given
handler and must be assessed separately.

Do not silently change emulator configuration or disable Fastmem,
caches, or other execution paths merely to increase log visibility. If a
setting must be changed for observability, explain the
behavioural/performance implications, verify that the path is actually
rerouted through the instrumented code, and treat the result as a
different test configuration.

Level 4 can generate enormous logs and materially affect timing. Warn
the user and keep it opt-in.

------------------------------------------------------------------------

## 4. Build the subsystem map before logging

Do not begin by placing `DEBUG_LOG` in every function. First inspect the
source and create a path inventory.

Use a table like this:

  -----------------------------------------------------------------------
  Layer                               Questions
  ----------------------------------- -----------------------------------
  Initialization/lifecycle            How is the subsystem created,
                                      reset, restored, and shut down?

  Public interface                    How does the CPU or another
                                      subsystem enter this code?

  Register/API interface              Which registers, commands,
                                      callbacks, or APIs arrive?

  Address decoding/mapping            How are addresses resolved,
                                      mirrored, masked, or mapped?

  Read paths                          What are the distinct read handlers
                                      and return paths?

  Write paths                         What are the distinct write
                                      handlers and side effects?

  Queue/buffer                        Where is data queued, peeked,
                                      consumed, rejected, or flushed?

  Decoder/dispatcher                  Where are commands or operation
                                      types identified and routed?

  Core operation                      Where does the actual work occur?

  State                               Which persistent fields affect
                                      later operations?

  Timing/events                       Which ticks, counters, events, and
                                      scheduling decisions matter?

  Memory/transfers                    Where are bytes, words, sectors, or
                                      pixels moved?

  Interrupts                          Where are requests raised,
                                      acknowledged, masked, or cleared?

  Backend/consumer                    Where is work handed to another
                                      subsystem, thread, renderer,
                                      device, or service?

  Completion/output                   Where can completion, failure,
                                      abort, or returned data be proven?

  Error/fallback                      What happens for invalid,
                                      unsupported, unmapped, or
                                      exceptional cases?
  -----------------------------------------------------------------------

Adapt the rows to the actual subsystem. Do not copy GPU-specific
categories into unrelated code.

### 4.1 Build a path checklist

For every important path, record:

-   Entry function.
-   Main branches or dispatch cases.
-   Data/state touched.
-   Exit or completion point.
-   Existing bypasses or alternate implementations.
-   Proposed trace level.
-   Whether the path is expected to occur in the test.

This checklist becomes the source-coverage audit.

### 4.2 Connect producer and consumer boundaries

A useful execution map connects both sides:

-   Input received → accepted/rejected.
-   Accepted → queued.
-   Queued → dequeued/consumed.
-   Decoded → dispatched.
-   Dispatched → executed.
-   Transfer started → data moved → transfer completed/aborted.
-   Interrupt raised → acknowledged/cleared.
-   Work submitted → backend processed/completed.

A start log alone does not prove that an operation completed.

### 4.3 Inspect bypasses and alternate execution paths

Explicitly search for paths that may bypass the function being
instrumented:

-   Fastmem or direct memory pointers.
-   Cached/recompiled CPU paths.
-   Direct register accessors.
-   DMA-specific paths.
-   Inline or templated accessors.
-   Backend/thread queues.
-   Direct memory copies or bulk operations.
-   Save-state restore paths.
-   Alternate handlers selected by configuration.

If a bypass is not instrumented, list it as a coverage limitation. Do
not imply that the primary handler sees it.

------------------------------------------------------------------------

## 5. Keep logging behaviour-neutral

### 5.1 Never evaluate a side-effecting expression twice

Unsafe:

``` cpp
DEBUG_LOG("result={}", ReadRegister());
return ReadRegister();
```

Safer:

``` cpp
const u32 result = ReadRegister();
TRACE_LOG(3, "result=0x{:08X}", result);
return result;
```

Only use a local value if it is the value already produced by the
original operation. Do not introduce an extra read to obtain it.

### 5.2 Preserve evaluation and ordering

Do not:

-   Call FIFO `Pop()` an extra time for logging.
-   Call a register getter again if it has side effects.
-   Synchronize timing an extra time.
-   Read memory the original code did not read.
-   Change the order of reads, writes, or state mutations.
-   Move a log across a mutation if that changes what the message
    represents.
-   Add locks, callbacks, or re-entrant calls to the subsystem.
-   Add dynamic allocation to a hot path without a strong reason.
-   Change return values, branch conditions, or error handling.

Log values already available in local variables or values produced by
the original operation.

### 5.3 Hot polling paths

Status registers, readiness checks, busy checks, timer reads, and small
timing queries may execute thousands or millions of times.

For Levels 1--3, prefer:

-   State transitions.
-   First occurrence plus change detection.
-   Per-transaction summaries.
-   Bounded samples.
-   Counts accumulated at a natural operation boundary.

Do not log every repeated poll at Level 1. If the read itself is
architecturally important, preserve evidence that the path exists; do
not simply omit the entire read path.

At Level 4, repeated reads may be logged only if the mode explicitly
promises exhaustive reads through that path. Keep this separate from the
normal concise levels.

------------------------------------------------------------------------

## 6. Bounded sampling: first five and last five

For large repetitive transfers, preserve:

-   Total actual item count.
-   First five items.
-   Last five items.
-   Index of each sample.
-   Transfer start/end metadata.
-   Expected count and actual count when meaningful.

Sample at the real data boundary: the point where the original code
already has the item that was actually accepted, returned, queued, or
consumed.

Do not confuse:

-   Requested items with accepted items.
-   Enqueued items with consumed items.
-   Raw data with decoded payload.
-   DMA-requested words with words accepted by the target.
-   A register access with an internal device operation.

Use fixed-size storage such as `std::array` where practical. Avoid
dynamic allocation in hot paths.

### 6.1 Correctness requirements

-   Maintain an accurate count of actual items observed.
-   Preserve chronological order in the last-five ring buffer.
-   Do not print duplicate entries when the transfer contains fewer than
    ten items.
-   Reset sample state at the correct transaction boundary.
-   Keep samples associated with the correct transaction if transfers
    can overlap or interleave.
-   Report aborted, interrupted, or incomplete transfers distinctly.
-   Do not add per-item log messages merely to maintain the sample
    buffer.

If the operation has no clear transaction boundary, do not invent one.
Explain the limitation and use an appropriate aggregation window.

### 6.2 Sampling is not exhaustive logging

A sampled RAM or register access path is visible, but it is not a full
transaction trace. The report must explicitly say that those accesses
are sampled.

------------------------------------------------------------------------

## 7. Decode bitfields and format values safely

DuckStation uses bitfield wrapper types in many registers. A
`BitField<Backing, Data, Index, Count>` is not necessarily a plain
integer member.

Rules:

-   Convert a field to its declared data type before formatting.
-   Prefer an explicitly typed local variable.
-   Do not assume the wrapper itself is accepted by `fmt`.
-   Do not take the address of a bitfield wrapper as if it were an
    ordinary integer.
-   Do not change register unions or bitfield layouts to make logging
    easier.
-   Do not modify `common/bitfield.h` to fix a logging expression.
-   Cast enums to a supported underlying/integer type when necessary.
-   Use the project's supported pointer formatter, such as `fmt::ptr()`,
    when appropriate.
-   Check the actual formatter support of custom, SIMD, vector, and
    template types.

Example:

``` cpp
const u16 display_end_x = register_fields.X2;
TRACE_LOG(2, "display_end_x={}", display_end_x);

const u32 raw_register = register_union.bits;
TRACE_LOG(2, "register=0x{:08X}", raw_register);
```

------------------------------------------------------------------------

## 8. Build errors in `base.h`, `bitfield.h`, or other shared headers

A compiler diagnostic pointing into a header does not prove that the
header is wrong. A new logging expression may trigger template
instantiation in `{fmt}`, type traits, enum formatters, SIMD formatters,
or overloaded operators.

When a build fails:

1.  Capture the complete build output.
2.  Find the first meaningful compiler error.
3.  Read the full template-instantiation chain.
4.  Find the first reference to the edited `.cpp`.
5.  Inspect every argument passed to that log statement.
6.  Check the actual type of each argument.
7.  Convert bitfields and enums to explicit supported types locally.
8.  Fix the logging expression in the edited `.cpp` where possible.
9.  Rebuild the same target and configuration.

Do not start by modifying `base.h`, `bitfield.h`, or third-party
formatting internals.

Do not silence errors with broad casts, remove qualifiers, disable
warnings globally, or replace meaningful types with `void*` merely to
make a message compile.

------------------------------------------------------------------------

## 9. Editing workflow

### Phase A --- Inspect

1.  Read the complete target `.cpp`.
2.  Read relevant headers and definitions.
3.  Inspect the supplied original and final reference source files.
4.  Identify all entry points, read/write paths, branches, handlers,
    queues, state, timing, and completion points.
5.  Identify hot paths and bypasses.
6.  Build the path checklist and subsystem flow map.
7.  Record existing log channels and conventions.

### Phase B --- Audit before editing

Create a coverage matrix:

  -----------------------------------------------------------------------------------
  Path/category   Source location   Current coverage    Planned level  Runtime test
  --------------- ----------------- ------------------- -------------- --------------
  Path A          Function/branch   None/partial/full   1/2/3/4        Yes/No

  Path B          Function/branch   None/partial/full   1/2/3/4        Yes/No
  -----------------------------------------------------------------------------------

Do not mark a path as covered merely because a nearby function logs.
Confirm that the relevant branch and both sides of the boundary are
represented.

### Phase C --- Plan

For each proposed log, decide:

-   What question does it answer?
-   Which level should contain it?
-   Is it a one-time boundary, transition, repeated poll, or bulk
    transfer?
-   Does it need aggregation or sampling?
-   What is the actual count being measured?
-   Can the value be obtained without another read or side effect?
-   Is the path inside this file's observation boundary?

Mark high-frequency paths as one of: - No log. - Transition-only. -
Aggregated count. - First/change/last sample. - Level 4 exhaustive, only
if explicitly requested.

### Phase D --- Instrument

1.  Add a local compile-time trace-level constant.
2.  Add a small trace macro using the existing log channel.
3.  Implement Level 1 coverage across all important paths first.
4.  Add Level 2 operation details.
5.  Add selective Level 3 diagnostics.
6.  Add bounded sampling where appropriate.
7.  Add Level 4 only if explicitly required.
8.  Preserve original source behaviour and ordering.
9.  Keep changes limited to the requested source file unless another
    file is genuinely required and the user approves.

### Phase E --- Static review

Check:

-   No original operation was removed or reordered.
-   No side-effecting expression is evaluated twice.
-   No extra FIFO pop, register read, memory access, or timing
    synchronization was added.
-   No register, memory, or emulated state is modified by diagnostics.
-   No return value, branch condition, or error path changed.
-   No hot path gained unbounded logging in Levels 1--3.
-   All important source paths have a deliberate coverage decision.
-   Samples, counts, and indices are correct.
-   Short transfers do not print duplicate samples.
-   Diagnostic state is reset correctly.
-   No unrelated headers or files were modified.
-   The edited file still follows project style and existing log
    conventions.

### Phase F --- Build

1.  Build the same target and configuration used for the original
    source.
2.  If it fails, diagnose the first meaningful error.
3.  Fix logging expressions locally before considering shared headers.
4.  Rebuild.
5.  Report one of: successful build, failed build, or build not
    attempted.

Static inspection is not a successful build.

### Phase G --- Runtime validation

Run a representative workload and check:

-   Important paths appear from entry to completion.
-   Level 1 is concise but does not omit important categories.
-   Level 2 explains operation handling.
-   Level 3 exposes useful internal evidence.
-   Level 4, if implemented, records every promised observable
    operation.
-   First/last samples are correct and chronological.
-   Counts match actual operations.
-   Repeated polls do not flood Levels 1--3.
-   Error, fallback, abort, and alternate paths are represented when
    exercised.
-   Emulator behaviour remains consistent with the uninstrumented
    version.

If possible, compare the same workload with tracing off and on. Heavy
logging can affect performance and scheduling; document that limitation.

------------------------------------------------------------------------

## 10. Reviewing the resulting log

Do not judge completeness by file size or line count alone.

Review:

1.  Can the main path be followed from input to completion?
2.  Are all important source-level paths represented?
3.  Are read and write paths both covered where applicable?
4.  Are producer and consumer boundaries connected?
5.  Are important state transitions visible?
6.  Are transfer counts and dimensions available?
7.  Are raw samples available where exact data matters?
8.  Are error, fallback, abort, and invalid paths visible?
9.  Are repeated logs hiding useful events?
10. Did the test exercise the relevant paths?
11. Are any operations bypassing the instrumented handlers?
12. Does the trace describe DuckStation's host-side implementation
    accurately, without confusing it with physical console hardware
    behaviour?

Keep these conclusions separate:

-   **Source instrumentation coverage:** functions and branches that
    contain instrumentation.
-   **Runtime execution coverage:** functions and branches actually
    observed in the test.
-   **Observation boundary:** operations this file can and cannot see.
-   **Trace completeness:** whether the chosen level is sampled,
    change-filtered, aggregated, or exhaustive.

A complete subsystem map requires source inspection and representative
runtime workloads. A single log cannot prove unexercised paths.

------------------------------------------------------------------------

## 11. Required response when editing a source file

When returning an edited source file, provide:

1.  **Subsystem map:** the important layers and execution paths found.
2.  **Coverage matrix:** covered, partial, missing, bypassed, and
    untested paths.
3.  **Level definitions:** exactly what Levels 1, 2, and 3 mean for this
    file; describe Level 4 separately if implemented.
4.  **Sampling plan:** what is counted, where sampling occurs, and
    whether data is sampled or exhaustive.
5.  **Excluded logging:** what was intentionally suppressed and why.
6.  **Build-risk notes:** bitfields, formatter types, inline functions,
    templates, timing-sensitive paths, and shared headers.
7.  **Complete edited `.cpp` file:** not snippets when a complete file
    was requested.
8.  **Change summary:** what was added and what remains outside scope.
9.  **Validation status:** static review, build, and runtime test
    reported separately.

Never say "full map" without qualifying the scope. State exactly which
paths are covered and what remains unobserved.

------------------------------------------------------------------------

## 12. Reusable request template

Use this template when instrumenting another DuckStation `.cpp` file:

> I have edited the attached DuckStation `[target.cpp]` from its
> original source.
>
> Please review and improve it so it provides a **complete important
> execution map of this subsystem**, following the DuckStation C++
> Source Instrumentation Skill.
>
> **First inspect the complete source and relevant headers. Do not start
> editing immediately.** Build a source-level map of all important entry
> points, read/write paths, dispatch branches, state transitions,
> transfers, timing events, completion paths, errors, and bypasses.
>
> Use three compile-time logging levels:
>
> -   **Level 1 --- Complete important execution map:** cover every
>     important execution path and major boundary, but do not log every
>     repeated poll or helper call.
> -   **Level 2 --- Detailed operation trace:** explain how each
>     important operation processes its inputs, decodes fields, moves
>     data, and changes state.
> -   **Level 3 --- Deep internal diagnostics:** provide selected raw
>     values, bitfields, internal state, timing details, and
>     first-five/last-five samples.
>
> For large repetitive data, log the actual total count plus the first
> five and last five items, with indices. Do not duplicate samples for
> short transfers. Make clear whether a path is sampled,
> change-filtered, aggregated, or exhaustive.
>
> Do not remove an important read/write path simply because it is
> repetitive. In Levels 1--3, preserve its visibility using meaningful
> boundaries, state changes, or counts. Do not log every unchanged poll
> unless Level 4 is specifically requested.
>
> **Optional Level 4:** if I explicitly request forensic tracing, record
> every observable operation routed through the instrumented paths,
> without sampling or change-only suppression for those paths. Clearly
> document bypasses and operations that this `.cpp` cannot observe. Do
> not change Fastmem, cache, timing, or other emulator settings without
> explaining and verifying the consequences.
>
> Preserve original emulator behaviour. Do not add extra hardware reads,
> FIFO operations, memory accesses, or timing synchronization for
> logging. Do not evaluate side-effecting expressions twice. Do not
> change algorithms, return values, branch conditions, or state layout.
>
> Refer to `base.h` and `bitfield.h` for type and formatting
> compatibility. Convert bitfields and enums to explicit supported types
> before formatting. If a build error points into a shared header, trace
> it back to the new logging expression first; do not modify shared or
> third-party headers just to make logging compile.
>
> Use:
>
> -   `[original target.cpp]` as the behavioural reference.
> -   `[final reference instrumentation.cpp]` as the quality and
>     structure reference.
> -   `DuckStation_CPP_Instrumentation_Skill.md` as the instrumentation
>     standard.
>
> Adapt the reference structure to the target subsystem. Do not blindly
> copy subsystem-specific logging.
>
> Before editing, report the source coverage plan and anything the
> target file cannot observe. After editing, provide the complete `.cpp`
> file and a concise explanation of covered paths, missing paths,
> sampling, excluded repetitive logs, and validation status.
>
> Distinguish static review, successful compilation, and runtime
> validation. Do not claim a build or test passed unless it was actually
> performed.

------------------------------------------------------------------------

## 13. Final principle

**Map every important path. Explain each operation. Sample repetitive
data. Keep exhaustive tracing separate. Preserve the machine being
observed.**

The best trace is not the one with the most messages. It is the one that
lets a reader reconstruct the important execution, understand the data
and state transformations, identify missing coverage, and know exactly
what the log cannot prove.
