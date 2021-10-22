# OCaml 5.0 Design

###### tags: `OCaml5.0`

_The Multicore Dev Team_

The aim of the document is to help review OCaml 5.0 changes as part of the upstreaming process [^upstream]. The proposed changes in 5.0 have been split into 5 working groups (WG) for easier review -- GC, Domains, Runtime multi-domain safety, Stdlib changes, Fibers. This document includes design notes for the working groups.

**Note:** In the rest of the document, the term Multicore OCaml and OCaml 5.00 are used interchangably.

## Getting started

The OCaml 5.00 development branch is at https://github.com/ocaml-multicore/ocaml-multicore/tree/5.00. The 5.00 tree is based off the _anchor commit_ [`6e053e0dbb`](https://github.com/ocaml/ocaml/commit/6e053e0dbb) (as of 22/10/2021).  The ocaml-multicore [PR#710](https://github.com/ocaml-multicore/ocaml-multicore/pull/710) shows the diff against the anchor commit. Given that the diff is large, it will be easier to use command-line tools to review the work. For example,

```bash=
git diff 6e053e0dbb --stat
```

shows the diff stats. 

```bash=
git diff 6e053e0dbb -- lambda
```

shows the diff for the `lambda` directory.

A project board for getting 5.00 ready for review is available here: https://github.com/ocaml-multicore/ocaml-multicore/projects/5.

### Communication

For the asynchronous review period, we suggest the use of email for communication. Please email the appropriate [working group leads](https://hackmd.io/0HZanf5ST2OBW10eyZxP_Q?both#Working-groups) who will include the relevant folks for the discussions. Alternatively, you can describe issues at [ocaml-multicore](https://github.com/ocaml-multicore/ocaml-multicore/) or send PRs to the [ocaml-multicore#5.00](https://github.com/ocaml-multicore/ocaml-multicore/tree/5.00) branch.

## Working Groups

### WG1: GC

The multicore runtime alters the memory allocation and garbage collector to support multiple parallel threads of OCaml execution. It utilizes a stop-the-world parallel minor collector, a StreamFlow like multithreaded allocator and a mostly-concurrent major collector; the design rational and details have been published previously[^icfp_gc_paper]. The garbage collector supports OCaml finalisers, ephemerons and the existing C-API. A particular challenge is ensuring that the per domain garbage collection structures are correctly cleaned up on domain termination.

The reviewer should approach the code by looking at the following areas:
 - `runtime/minor_gc.c`: contains the stop-the-world minor collector
 - `runtime/major_gc.c`: contains the major collector
 - `runtime/shared_heap.c`: contains the pool based memory allocator
 - `runtime/finalise.c`: is altered to support multiple domains; care is also needed to handle the handoff of finalisers on domain termination
 - `runtime/weak.c`: implements the ephemeron support
 - `runtime/globroots.c`: is altered to ensure global roots are correctly handled by the multicore GC

There is potential for further improvement after the MVP is merged: 
 - there is no compaction. The pool allocation strategy used in the shared heap relieves some need for compaction. However implementing compaction would be non-trivial and require some research work;
 - currently mark stacks can be unbounded to avoid remarking. The solution presented in ocaml-multicore#474[^ocaml-multicore#474] is complex and a stop-the-world implementation might be better;
 - optimizations to the distribution of global root marking and load balancing of the mark stack work between domains; 
 - mark stack prefetching has not been implemented in multicore;
 - statmemprof will need to be integrated onto the multicore GC system.

### WG2: Domains

Each domain in multicore can execute a thread of OCaml in parallel with other domains. Several additions are made to OCaml to spawn new domains, join domains that are terminating and provide domain local storage. There is a stdlib module `Domain` and the underlying runtime domain structures. 

The challenge for the runtime structures is to accurately maintain the set of domains that must take part in stop-the-world sections in the presence of domain termination and spawning, as well as ensuring that a domain services stop-the-world requests when the main mutator is in a blocking call; this is handled using a _backup thread_ signaled from `caml_enter_blocking_section` / `caml_leave_blocking_section`. 

The reviewer should approach the code by looking at the following areas:
- `runtime/domain.c`: contains the code for starting a new domain, the handling of domain termination,  joining with other domains, the API for other runtime components to initiate stop-the-world sections and the backup thread. 
- `stdlib/domain.ml`: the OCaml interface to control creation and joining of domains.
- `lambda` and `asmcomp`: to thread through `dls_get` allowing efficient retrieval of domain-local variables  


### WG3: Runtime multi-domain safety

#### Systhreads

Multicore OCaml supports systhreads in a backwards compatible fashion. The execution model remains the same, except transposed to domains rather than a single execution context.

Each domain will get its own threads chaining: this means that while only one systhread can execute at a time on a single domain (akin to trunk), many domains can still execute in parallel, with their systhreads chaining being independent. To achieve this, a thread table is employed to allow each domains to maintain their own independent chaining. Context switching now involves extra care to handle the backup thread. The backup thread takes care of GC duties when a thread is currently in a blocking section. Systhreads needs to be careful about when to signal it.

The tick thread, used to periodically force thread preemption, has been updated to not rely on signals (as the multicore signaling model does not allow this to be done efficiently). Instead, we rely on the interrupt infrastructure of the multicore runtime and trigger an "external" interrupt, that will call back into systhreads to force a yield.

The following sections of the codebase are relevant to this change:

- `otherlibs/systhreads/st_stubs.c`
- `otherlibs/systhreads/st_posix.h`

#### codefrag

Code fragments are now stored in a lockfree skiplist to allow multiple threads to work on the codefrags structures concurrently in a thread-safe manner. Extra care is required on cleanup (i.e, freeing unused code fragments entries): this should only happen on one domain, and this is done at the end of a major cycle (behind `caml_global_barrier_is_final`).

The following sections of the codebase are relevant to this change:

- `runtime/codefrag.c`: is updated to use the new lockfree skiplist
- `runtime/lf_skiplist.c`: the actual implementation of this skiplist
- `runtime/major_gc.c`: aforementionned garbage cleanup is done at the end of a major cycle, in `cycle_all_domains_callback`.

[ocaml-multicore#672](https://github.com/ocaml-multicore/ocaml-multicore/pull/672) is also a recommended read, it contains an early review of the skiplist implementation and codefrag change by Luc Maranget.

#### Signals

Signals in multicore have the following behaviour:

* A program with a single domain should have the same signal behaviour as trunk (with the minor exception detailed later). This includes the delivery of signals to systhreads on that domain.
* Programs with multiple domains treat signals in a global fashion. It is not possible to direct signals to individual domains or threads, other than the control through thread sigmask. A domain recording a signal may not be the one executing the OCaml signal handler.

Signals in multicore introduces a slight semantic change compared to the current implementation found in `trunk`. The pending signals structure now holds atomic counters rather than holding a single flag per signal number. The thread receiving the signal will increment both the relevant signal counter and the pending signals counter. Signals are then processed later by the next thread hitting `caml_process_pending_signals`. The thread will then process all the pending signals (other the signals that are blocked by this thread).

The slight semantics change between trunk and our implementation lies in the fact that in multicore, if signals are sent in a burst, the runtime may process as many signals as received (because received pending signals are tracked through atomic counters). In trunk, a similar burst of signals may end up triggering a signal handler only once.

There is currently an outstanding issue with signals that is being worked on: https://github.com/ocaml-multicore/ocaml-multicore/issues/703

The following sections of the code base are relevant to this change:

- `runtime/signals.c`

#### extern/intern

Externing and interning are made domain-local in the multicore runtime by storing their respective states in the `domain_state.` The relevant files are:

- `runtime/intern.c`
- `runtime/extern.c`
    
The semantics of marshaling an object that is concurrently mutated by another domain is undefined and may lead to corrupted marshaled object. 

#### Frame descriptors

Frame descriptors modifications are now locked behind a mutex to avoid races if different threads were to try to apply changes to the frame table at the same time. Freeing up old frame tables is done at the end of a major cycle (which is a STW section) in order to be sure that no thread will be using this old frame table anymore.

Dynlink is not adjusted to be thread-safe. The API for Dynlink is very stateful. In particular, the access control operations affect the behaviour of later call to load a file. Concurrently setting the access control permissions and loading a file is undefined.

The following sections of the codebase are relevant to this change:

- `runtime/frame_descriptors.c`: this now contains most of the code related to frame descriptors and frametables handling.
- `runtime/backtrace_nat.c`

#### Eventlog

Multicore contains a version of eventlog that is safe for multiple domains. It achieves this by having a separate CTF file per domain but this is an interim solution. We hope to replace this implementation with an existing prorotype based on per-domain ring buffers which can be consumed programmatically from both OCaml and C.

The new implementation avoids the performance issues that required putting eventlog in the instrumented runtime. There is an in-progress tree for this work that we hope to merge in the next few weeks: https://github.com/sadiqj/ocaml-multicore/tree/eventring_multicore

### WG4: Stdlib changes

The main guiding principle in porting the Stdlib to OCaml 5.00 is that 

1. OCaml 5.00 does not provide thread-safety by default for mutable data structures and interfaces.
2. OCaml 5.00 does ensure memory-safety (no crashes) even when stdlib is used in parallel by multiple domains.
3. Observationally pure interfaces remain so in OCaml 5.00.

The motivation for these choices and further discussion is found in the Multicore OCaml wiki page [^stdlib_safety].

#### Lazy

Lazy values in OCaml allow deferred computations to be run by _forcing_ them. Once the lazy computation runs to completion, the lazy is updated such that further forcing fetches the result from the previous forcing. The minor GC also short-circuits forced lazy values avoiding a hop through the lazy object. The implementation of lazy uses [unsafe operations from the Obj module](https://github.com/ocaml/ocaml/blob/trunk/stdlib/camlinternalLazy.ml).

The implementation of Lazy has been made thread-safe in OCaml 5.00. 

For single-threaded use, the Lazy module preserves backwards compatibility. For multi-threaded use, the Lazy module adds synchronization such that on concurrent forcing of an unforced lazy value from multiple domains, one of the domains will get to run the deferred computation while the other will get a new exception `RacyLazy`.

For the new semantics, consult the documentation in `stdlib/lazy.mli`. For the implementation details, have a look at

* `stdlib/lazy.ml`
* `stdlib/camlinternalLazy.ml`

#### Random

With [ocaml-multicore#582](https://github.com/ocaml-multicore/ocaml-multicore/pull/582), we have domain-local PRNGs following closely along the lines of stock OCaml. In particular, the behaviour remains the same for sequential OCaml programs. But the situation for parallel programs is not ideal. Without explicit initialisation, all the domains will draw the same initial sequence.

There is ongoing discussion on splittable PRNGs in [ocaml/RFCs#28](https://github.com/ocaml/RFCs/pull/28), and a re-implementation of Random using the Xoshiro256++ PRNG in [ocaml/ocaml#10701](https://github.com/ocaml/ocaml/pull/10701). 

#### Format

The Format module has some hidden global state for implementing pretty-printing boxes. While the module has explicit API for passing the formatter state to the functions, there are predefined formatters for `stdout`, `stderr` and standard buffer, whose state is maintained by the module.

The Format module has been made thread-safe for predefined formatters. We use domain-local versions of formatter state for each domain, lazily switching to this version when the first-domain is spawned. This preserves the performance of single-threaded code, while being thread-safe for multi-threaded use case. See the discussion in [ocaml/ocaml#10453](https://github.com/ocaml/ocaml/issues/10453#issuecomment-868940501) for a summary. 

#### Mutex, Condition, Semaphore

The Mutex, Condition and Semaphore modules are the same as systhreads in stock OCaml. They now reside in `stdlib`. When systhreads are linked, the same modules are used for synchronization between systhreads.

#### Atomics

The OCaml memory model offers a modular version of data-race-freedom guarantee called "local data-race-freedom", which bounds the effect of racy accesses in space and time. OCaml memory model has been published in the PLDI 2018 paper [^memory_model]. The paper was also covered by The Morning Paper in two parts -- [part I](https://blog.acolyer.org/2018/08/09/bounding-data-races-in-space-and-time-part-i/) and [part II](https://blog.acolyer.org/2018/08/10/bounding-data-races-in-space-and-time-part-ii/).

The paper shows the compilation to x86 and ARMv8. The memory model is "free" for x86 -- the non-atomic loads and stores are compiled to plain loads and stores on x86. The module `Atomic` in stdlib provides the new atomic primitives. The primitives in the module are implemented in `runtime/memory.c`. We use C11 atomics to enforce the memory model in `runtime/memory.c`. We also have a [herd formalisation of the memory model](https://github.com/ocamllabs/ocaml-memory-model).

For the OCaml 5.00 MVP, the only architecture that is support is x86. The real work for compilation of memory model will be for architectures with weaker ordering such as Arm and Power. We had done some experiments on sequential OCaml for evaluating the cost of enforcing the memory model for the PLDI 2018 paper. The experimental branches are:

* branch-after-load-aarch64: https://github.com/kayceesrk/ocaml/tree/branch-after-load-aarch64 (compilation scheme 1 in Table 5 of the paper)
* dmb-before-store-aarch64: https://github.com/kayceesrk/ocaml/tree/dmb-before-store-aarch64 (compilation scheme 2 in Table 5 of the paper)

### WG5: Fibers

Fibers are the runtime system mechanism that supports effect handlers. The design of effect handlers in OCaml has been written up in the PLDI 2021 paper [^retro_concurrency]. It will be useful to skim through the paper before continuing with the rest of this section. The motivation for adding effect handlers and some more examples are found in these slides [^effects_talk]. 

#### Programming with effect handlers

Effect handlers are made available to the OCaml programmer from `stdlib/effectHandlers.ml`. The EffectHandlers module exposes two variants of effect handlers -- deep and shallow. Deep handlers are like folds over the computation tree whereas shallow handlers are akin to case splits [^dhil_jfp]. With deep handlers, the handler wraps around the continuation, whereas in shallow handlers it doesn't. 

Here is an example of a program that uses deep handlers to model something analogous to the `Reader` monad. 

```ocaml=
open EffectHandlers
open EffectHandlers.Deep

type _ eff += Ask : int eff

let main () =
  try_with (fun _ -> perform Ask + perform Ask) ()
  { effc = fun (type a) (e : a eff) ->
      match e with
      | Ask -> Some (fun (k : (a,_) continuation) -> continue k 1)
      | _ -> None }

let _ = assert (main () = 2)
```

Observe that when we resume the continuation `k`, the subsequent effects performed by the computation are also handled by the same handler. As opposed to this, for the shallow handler doesn't. For shallow handlers, we use `continue_with` instead of continue.

```ocaml=
open EffectHandlers
open EffectHandlers.Shallow

type _ eff += Ask : int eff

let main () =
  let rec loop (k: (int,_) continuation) (state : int) =
    continue_with k state
    { retc = (fun v -> v);
      exnc = (fun e -> raise e);
      effc = fun (type a) (e : a eff) ->
        match e with
        | Ask -> Some (fun (k : (a, _) continuation) -> loop k 1)
        | _ -> None }
  in
  let k = fiber (fun _ -> perform Ask + perform Ask) in
  loop k 1

let _ = assert (main () = 2)
```

Observe that with a shallow handler, the recursion is explicit. Shallow handlers makes it easier to encode cases where state needs to be threaded through. For example, here is a variant of the `State` handler that encodes a counter:

```ocaml=
open EffectHandlers
open EffectHandlers.Shallow

type _ eff += Next : int eff

let main () =
  let rec loop (k: (int,_) continuation) (state : int) =
    continue_with k state
    { retc = (fun v -> v);
      exnc = (fun e -> raise e);
      effc = fun (type a) (e : a eff) ->
        match e with
        | Next -> Some (fun (k : (a, _) continuation) -> loop k (state + 1))
        | _ -> None }
  in
  let k = fiber (fun _ -> perform Next + perform Next) in
  loop k 0

let _ = assert (main () = 3)
```

While this encoding is possible with deep handlers (by the usual `State` monad trick of building up a computation using a closure), it feels more natural with shallow handlers. In general, one can easily encode deep handlers using shallow handlers, but going the other way is challenging. With the typed effects work currently in development, the default would be shallow handlers and deep handlers would be encoded using the shallow handlers. 

As a bit of history, the current implementation is tuned for deep handlers and has gathered optimizations over several iterations. If shallow handlers becomes more widely in the coming years, it may be possible to put in some tweaks that removes a few allocations. That said, the semantics of the deep and shallow handlers in this future implementation will remain the same as what is currently in OCaml 5.00 branch. 

#### Navigating the code

The following files include the major changes that support effect handlers in OCaml 5.00.

* `stdlib/effectHandlers.ml` -- The user interface to effect handlers. Start here. The file introduces a few new primitives `%perform`, `%runstack`, `%resume` and `%reperform`. These are implemented in `runtime/interp.c` for the bytecode backend and `runtime/amd64.S` for the native code. Other than these, there are additional primitives implemented as C functions in `fiber.c`.
* `testsuite/tests/effects/*` -- Lots of tests for effect handlers. More tests are in `testsuite/tests/parallel/*`, `testsuite/tests/backtrace/*`.
* `runtime/interp.c` -- The bytecode implementation of fiber support.
* `asmcomp/amd64/emit.mlp` -- The stack overflow check for OCaml functions.
* `runtime/amd64.S` -- Native-implementation of fiber support. Most of the action is here.
* `runtime/fiber.c` -- Support for fiber allocation, freeing and GC interaction.

#### Stack Layout

The biggest change in the implementation of fibers in OCaml 5.00 is the stack layout. While OCaml uses the system stack for OCaml functions, OCaml 5.00 manages its own stack in the runtime. The layout of the stack can be found in `runtime/caml/fiber.h`.

The stacks are malloc'ed when the program first enters OCaml (`caml_alloc_main_stack` in `fiber.c`) and when effect handlers are installed (`caml_alloc_stack`). The stacks for effect handlers start out really small (32 words without the metadata) in order to keep the cost of effect handlers low. These stacks are freed when the computations within the effect handlers returns. We use `malloc` and `free` to manage the fiber stacks. There is a stack cache of recently freed stacks, which speeds up the stack allocations further.

The use of runtime managed stacks has two main implications:

1. OCaml code needs stack overflow checks.
2. External calls need to be executed on the system stack.

##### Stack overflow checks

The stack overflow checks are implemented in the `asmcomp/amd64/emit.mlp`. The checks are inserted in the function prologue. As an optimisation for tail functions, there is a small red zone at the top of the fiber stack space (16 words). If the tail functions have a frame size that's smaller than the red zone, then the stack overflow check is elided. See the [^retro_concurrency] paper for the performance impact of the red zone.

The stack layout does not change how exceptions are implemented in OCaml. However when stacks are reallocated, we need to walk the exception chain to update the absolute exception addresses to reflect the new stack location. A previous implementation of multicore OCaml changed the exceptions represented as offsets within the stack; this implementation made stack reallocation fast since no rewriting was done, however it altered all OCaml exceptions in a way that was highly detrimental to performance of some code using exceptions for flow control.[^ocaml-multicore#345] 

##### External calls

Given that we do not have stack overflow checks for C functions, we will need to switch to the system stack for performing C calls. For details, start at `Iextcall` handling in `asmcomp/amd64/emit.mlp`. 

Each domain has a system stack pointer `c_stack` in the domain state. This represents the position at which control switched to the OCaml stack. On external calls, we set `%rsp` to this position to make external calls. For external calls where arguments may be passed on the stack, these arguments need to be copied over to the C stack. The assembler function `caml_c_call_stack_args` achieves this. 

##### Callbacks

Since callbacks may be frequent, especially for things such as finalisers, we execute the callbacks in the stack space of the last fiber that was in use. Hence, there may be more than one logical, contiguous OCaml stack segment placed contiguously on the physical runtime managed OCaml stack (not too different from what happens with external calls).

#### DWARF debugging support

Given that the program stack is now split across the c_stack, and multiple runtime managed OCaml stacks, we introduce additional support for DWARF backtraces. In particular, `struct c_stack_link` in `runtime/caml/fiber.h` exists only for the purpose of DWARF backtraces. This structure is allocated in `caml_start_program` on the C stack. At `caml_c_call*`, we update this structure with information necessary for DWARF unwinding. See `runtime/amd64.S` for the new DWARF bytecode sequences (look for `.cfi_escape`) to unwinding across multiple physical stacks. 

#### GC interaction

For GC interaction, please see section 5.4 in the ICFP 2020 paper [^icfp_gc_paper]. For the corresponding code, see `caml_darken_cont` in `runtime/major_gc.c`.

## Not in MVP

* All architectures except x86 -- The only supported platforms are x86 on Linux, MacOS and Windows (mingw-w64).
* ocamldebug
* statmemprof
* AFL support  -- https://github.com/ocaml-multicore/ocaml-multicore/issues/497
* Compilation with frame-pointers
* Ephemeron C API -- There was discussion in the Q1 2021 caml-devel meeeting about better support for weak sets and hash table in the runtime. 


[^dhil_jfp]: Hillerstrom et al., "Effect Handlers via Generalised continuations", https://www.dhil.net/research/papers/generalised_continuations-jfp-draft.pdf

[^effects_talk]: Effect Handlers in Multicore OCaml, https://speakerdeck.com/kayceesrk/effect-handlers-in-multicore-ocaml

[^icfp_gc_paper]: Sivaramakrishnan et al., Retrofitting Parallelism onto OCaml, https://arxiv.org/abs/2004.11663

[^memory_model]: Dolan et al., Bounding data races in space and time https://dl.acm.org/doi/10.1145/3192366.3192421

[^ocaml-multicore#345]: Absolute exception stack, https://github.com/ocaml-multicore/ocaml-multicore/pull/345

[^ocaml-multicore#474]: Fixing remarking to be safe with parallel domains, https://github.com/ocaml-multicore/ocaml-multicore/pull/474

[^retro_concurrency]: Sivaramakrishnan et al., Retrofitting effect handlers onto OCaml, https://arxiv.org/abs/2104.00250

[^upstream]: OCaml 5.0 Upstreaming plan: https://hackmd.io/0HZanf5ST2OBW10eyZxP_Q?view

[^stdlib_safety]: Safety of Stdlib under Multicore OCaml, https://github.com/ocaml-multicore/ocaml-multicore/wiki/Safety-of-Stdlib-under-Multicore-OCaml