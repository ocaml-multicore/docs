# A proposal for upstreaming multicore to OCaml 5.0

###### tags: `OCaml5.0`

*The Multicore Dev Team*

Since the rebase of multicore to OCaml 4.12, the multicore team has been focusing on squeezing out bugs, improving API compatibility with existing OCaml code, simplifying the multicore runtime implementation and reducing the diff with upstream OCaml. We believe most of the major design decisions are now behind us for a minimum-viable-product (MVP). It is time to figure out how to merge the remaining pieces.

Following discussions with the OCaml core developers, it has been agreed that OCaml 5.0 will include the support for parallelism and the runtime system support for effect handlers (fibers). This is different from the earlier decision to include only the support for parallelism without the changes for fibers, which was slated to land in 5.1. Given that the addition of fibers to OCaml is not a user-observable change, the plan is to push all the multicore-related runtime system changes in one go. Thus, in OCaml 5.0, there will be no syntactic support for programming with effect handlers, but handler programs can be written using [stdlib functions](https://github.com/ocaml-multicore/ocaml-multicore/pull/616).

Multicore team believes that incremental PRs of multicore components to upstream OCaml, reviewed solely on Github, is hard. Many of the multicore components interlock, and individual components will introduce (minor) slowdowns without a corresponding increase in user functionality. Testing of many components only make sense with the presence of parallel domains. 

For example, consider a separate PR to OCaml which adds the new multicore-capable allocator alone. Given that the purpose of the new allocator is to support multiple domains running on top, not all of the design decisions will make sense by looking at the PR. Moreover, there is no clear way to get a good coverage of the new allocator without the ability to create multiple domains. Also, splitting out such components cleanly is a great deal of work for the multicore team, and the effort may not be worth it if the individual components are going to be only used together eventually. There is also the risk of introducing bugs in an effort to refactor the components.

*Our proposal is a multi-staged review process involving the multicore team and the OCaml dev team so that the parallelism and the runtime system support for concurrency (fibers) in Multicore OCaml can be merged as a **single PR**.* The syntactic support for effect handlers is proposed to be added together with an effect system at a later stage. 

The rest of the document contains the details of the proposal to upstream multicore support towards an OCaml 5.0 release.

## Where we are

Here is a summary of the status of Multicore OCaml:

* The current multicore development branch is`4.12+domains+effects`. This branch includes the effects syntax changes and so can break some code (e.g. the effect keyword) but also has ppx implications.
* `4.12+domains` branch removes the effects syntax and all API observable changes for fibers & effects. This branch does currently include the runtime and assembler changes for fibers. 
* The Multicore OCaml GC paper published at ICFP 2020 [3] includes results that were obtained on `4.12+domains+effects`, which includes the overhead of the runtime system changes for effect handlers. Moreover, the details of the overheads of fibers on programs that do not use fibers are found in section 6.1 of the PLDI 2021 paper [2]. Again, the results in the PLDI paper were obtained on `4.12+domains+effects`, and include the multicore GC changes.
* There have been significant QA and debugging efforts using [opam-health-check](http://check.ocamllabs.io:8082) which runs an opam universe and ocaml-multicore-ci which runs the testsuite of select packages (Batteries, Core, Async, LWT, Coq, Irmin). This testing is squeezing out API incompatability (which now seems to center on libraries directly using GC colors and specifics of the GC) and gets us confident the multicore implementation contains fewer bugs. Importantly these CI tools can be reused on OCaml 5.0 pre-release trees.
* On the development side, multicore has been improved in the following ways:
  + stdlib has been made multi-domain safe: https://github.com/ocaml-multicore/ocaml-multicore/wiki/Safety-of-Stdlib-under-Multicore-OCaml
  + We have simplified the proposed Domain API to essentially spawn, join and domain-local-storage. At the same time we have greatly simplified the runtime implementation of domains to remove historical artifacts and also the need for any domain-to-domain communication outside stop-the-world interrupts.
  + `intern` and `extern` have been made multi-domain safe (and rebased to the upstream OCaml implementation)
  + We have tried to clean up various (unnecessary) divergences between multicore and stock OCaml.
  + Current ongoing items we expect to complete by September are: signal handling brought into line with upstream and made multi-domain safe; codefrag is becoming multi-domain safe.

Importantly the major design decisions in multicore have stablized and as such the multicore team believes things are ready; some these design decisions have been published formally [1,2,3].

## A proposal for what should happen next

We are keen to get to a branch that can become the OCaml 5.0 release sooner than later. To do this we need to somehow merge the multicore domains and fibers functionality upstream in a timely and efficient fashion. The purpose is to get us to an MVP that can be iterated upon, not something that can be immediately released as OCaml 5.0.

The remaining items to upstream from multicore are highly interlocking and also quite large in nature. We propose thinking of the changes in 5 groups:

* **GC:** minor GC, major GC and the allocator
* **Domains:** spawn, join and domain-local-storage
* **Runtime multi-domain safety:** intern, extern, codefrag, dynlink, systhreads, signals
* **Stdlib changes:** changes for multi-domain safety, changes to
 `Lazy` and the public API for domain spawn along with domain-local-storage
* **Fibers:** The changes to the stack layout, FFI (non-user-observable), and the GC to support tracing the new stack layout.

### Working groups 

We propose a lead each from the multicore team and the OCaml dev team for each of the working groups who will shepherd the discussion. The proposed leads are:

| Working Group               | Multicore Lead           | Dev Team Lead                         |
| --------------------------- | ------------------------ |:------------------------------------- |
| GC                          | Tom Kelly & Sadiq Jaffer | Damien Doligez                        |
| Domains                     | Tom Kelly                | TBD                                   |
| Runtime multi-domain safety | Enguerrand Decorne       | Xavier Leroy                          |
| Stdlib changes              | KC Sivaramakrishnan      | Florian Angeletti (and Gabriel Scherer) |
| Fibers                      | KC Sivaramakrishnan      | Damien Doligez (and Xavier Leroy)          |

GC is the biggest change and we anticipate Tom and Sadiq to take lead on different sub-items.

We propose a working-group process for each area to look like this:

* **Stage 1 (Sep):** The multicore team will first finish the outstanding pieces for the MVP (tracked in a [Github project tab](https://github.com/ocaml-multicore/ocaml-multicore/projects/4)). We will then promote the multicore tree (currently at 4.12) to trunk OCaml. 
* **Stage 2 (Oct):** The multicore team will prepare a diff for each of the given area. This includes a (short) design document outlining the changes along with a guide to the diff itself. This will be wrapped up as a 30 min presentation for each area that we will present to the dev team. The working group for that area will then provide an initial high level review on any structural level concerns or areas of testing concern.
* **Stage 3 (Oct/Nov):** The multicore team will incorporate and respond to this initial review feedback. The aim is to prepare things for the in-person/dedicated sprint review.
* **Stage 4 (Nov):** The working groups for an area meet concurrently **over a week** with the multicore team to iterate the whole tree (and so the whole diff) to something that can be merged. At the end of that week, there may be further niggles and sweep up work.
* **Stage 5 (Nov/Dec):** The multicore team gets the niggles and further work sorted out.
* **Stage 6 (Dec):** After each of the working groups have given their green light, **a single PR against trunk** that can go through public reviews for each of the focus areas. The PR will also include the summaries from the working groups so that context may be provided for design decisions. The reviews at Stage 6 (Github PR stage) are not intended to lead to design debate (however if major design errors are discovered that would of course be a problem!), rather the PR provides a mechanism for the wider community as well as the core devs to submit any feedback and provide a mechanism for final approval.
* **Stage 7 (Jan/Feb):** Multicore team will iterate over the proposed changes. The compiler release readiness team at OCaml Labs will work with the dev team to perform final checks before merging. At this point, the multicore tree becomes OCaml trunk and the commit before the merge is tagged as a `LTS` version. Ideally, only bug fixes go into the LTS version, and the dev team works to ensure that missing features are implemented.

### The pros of going this way:

+ Maintain momentum and potentially quicker to merge
+ Much less work trying to track moving upstream target as window that multicore *hovers* over trunk is shorter
+ Might be more fun to have a concerted big heave to get multicore over the line
+ Likely much more efficient process than trying to use Github on large interlocking PRs

### The cons of going this way:

+ Significant block out of multicore team to deliver in short period
+ Significant time commitment outside the multicore team particularly for the review sprint in stage 4
+ How to ensure community is engaged and brought along with the changes
+ Maybe we want to make this take a long time as it might cut mistakes (or not).

### What's not in MVP

For clarity, there will be some significant features that have not been deemed to be minimum-viable-product in this above sprint:

+ Linux, OS X, Windows (mingw-w64 only)
+ AMD64 support only (ARM64 and others to follow later)
+ statmemprof
+ Flambda integration
+ Compaction

The idea would be that once the 5.0 branch is started, the standard open-source PR process could resume to fill out and improve things: e.g. ARM support, Windows support, PowerPC, flambda, etc.

### Open questions

* What tooling should we be using to iterate the proposed tree during the working group stages? (a Github branch in the multicore repo?)
* How are we going to efficiently do changes on the stack of interlocking PRs at the end?

## What next?

The next steps look like:

+ We discuss this plan in the OCaml dev meeting to see if the above process for review and merging is deliverable (of if they have a radically better way to do this).
+ We start getting a timescale together and set up who is in which working groups on the multicore and OCaml dev teams.

## References

[1] Bounding Data Races in Space and Time https://dl.acm.org/doi/10.1145/3192366.3192421
[2] Retrofitting Parallelism onto OCaml https://dl.acm.org/doi/10.1145/3408995
[3] Retrofitting Effect Handlers onto OCaml https://dl.acm.org/doi/10.1145/3453483.3454039