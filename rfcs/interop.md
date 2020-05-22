# Towards better systems programming in OCaml with out-of-heap allocation

The current multicore OCaml implementation bans so-called “naked
pointers”, pointers to outside the OCaml heap unless they follow
drastic restrictions. A backwards-incompatible change has been
proposed to make way for the new multicore GC in OCaml. I argue that
out-of-heap pointers are not an anomaly, but are part of a better
systems programming future, and I propose to show that the address
space reservation technique makes them compatible with the multicore
GC.

This is inspired by @jhjourdan's experiment with a contiguous address
space at <https://github.com/ocaml/ocaml/issues/6101>. Seven years
later, reserving address space to efficiently recognise in-heap
pointers (and do even wilder tricks) is standard in garbage collected
languages and allocators, and in research about memory management. In
addition, allowing foreign pointers for interoperability is a standard
feature in common systems programming languages.

As part of my research to make OCaml a better systems programming
language, I am planning to revisit this approach. I am joined in this
exploration by @rmdouglas, who worked in the industry on a project
which involved Ancient-like allocation for a shared heap in OCaml and
who is interested in this project. I explain the current plan below, I
would be very thankful if we could get some feedback before we dig
further.

This particular project is recent, although I have been interested in
off-heap allocation for a while as part of my research. Indeed, I had
the occasion to ask a few OCaml devs a few months ago, who gave some
reasons as to why the contiguous heap approach would be bad (and so
why it was abandoned, I understood). But when reading @kayceesrk's et
al.'s “Retrofitting parallelism onto OCaml”
(https://arxiv.org/abs/2004.11663) three weeks ago, I realised that
this technique is already used by the concurrent collector for the
minor heap–so it was hard to see which drawbacks there could remain in
practice. This realisation motivated me to have a second look and
learn all about this technique, and to investigate its supposed
drawbacks by myself. I now believe that the drawbacks are few or
non-existent, and I report some of my research below.

This draft RFC (“Request for comments”) is comprised of two parts:

- The proposal itself, of a path to keep supporting out-of-heap
  pointers. The reason why I decided to take advantage of the RFC
  repository is to offer a space for focused discussion with an
  evolving document that takes feedback into account. It is rendered
  here [TODO; currently next].

- Motivations given below and structured as follows:
  1. Gathered evidence from the practice and the literature
  2. Interest of out-of-heap allocation in current and future OCaml
  3. Dispelling common misconceptions about address space reservation

In particular, please inform us about any elements that we might be
missing and that have been used to justify the current
"no-naked-pointers" approach, that are not currently available
publicly.

This is part of a project aiming originally for the longer term, but
there is an obvious opportunity of _quickening the transition to
multicore_ by avoiding the breaking changes brought by the removal of
the page table, if we come to a conclusion on the proposed design
early enough. So please help us determine where the bar is placed.

# Efficient out-of-heap pointer detection

To efficiently recognise in-heap from off-heap pointers, the broad
idea is to offer a small *address space map* of very large reserved
address spaces. The address space is reserved as needed, similarly to
what is done in the Go language
[(details)](https://github.com/golang/go/blob/9acdc705e7d53a59b8418dd0beb570db35f7a744/src/runtime/malloc.go#L77-L99).

The proposed implementation work is to demonstrate and implement an
address space map for the non-multicore GC, starting from
Jacques-Henri Jourdan's original approach of reserving a single
contiguous virtual address space at
[#6101](https://github.com/ocaml/ocaml/issues/6101).

The criterion is that performance must be comparable to the
no-naked-pointers mode and not have extravagant adverse effect. This
is plausible thanks to the companion survey work and the original
experiment.

In addition, I have identified 4 challenges which must be convincingly
shown solvable to ensure that the result is both usable and scalable:

1. the “growing heap” issue, whereby out-of-heap pointers become
   in-heap after the heap grows,
2. 32-bit support & large memory support,
3. the cost of synchronisation for multicore,
4. malloc implementation for multicore.

## Language reference

This is the only "normative" section of this proposal.

1. Word-aligned out-of-heap pointers are valid OCaml values.

2. Out-of-heap pointers that enter the OCaml heap must be guaranteed
   to not belong to _any future_ reserved mapping of the OCaml heap.

## Explanation

The clause 1. preserves current behaviour. It is also important to
note that non-aligned out-of-heap pointers and out-of-heap pointers to
char are forbidden, or to explain to which extent they would be
allowed inside OCaml values (and operated with external primitives,
for instance), due to the proliferation of reliance on the
`0bxxxxxx10` bit pattern in the runtime to propagate exceptions.

The word of caution from the manual about out-of-heap pointers that
become in-heap pointers after the _extension du domaine de la_
collection is made normative as the clause 2. In other words, it is
left to the user to make sure that this does not happen, by whatever
means they have (but let us make sure that realistic context-specific
solutions exist). The reason for not mandating a specific solution
for 2. is that this is outside of the scope of OCaml: indeed, it
concerns the specific interoperation between OCaml and another
component. The mistake, I believe, is to look for a “silver bullet”
that solves the problem independently of the context.

## Challenge 1: the “growing heap” issue

> “Doctor, it hurts when I do this...
>
>  — Then don't do it!”

Current OCaml and OCaml multicore get memory chunks from the system
allocator. Consequently, the risk that out-of-heap pointers allocated
by malloc become in-heap after the memory has been freed is real. The
GC will be confused and follow these out-of-heap pointers, leading to
a crash.

In practice, the “growing heap” issue is less of a problem than it
sounds, at least when using address space reservation. One must be
“very unlucky” if a full huge mappable address range becomes entirely
available (unreserved) after some memory was allocated inside of it,
at a location close to where we choose to extend the address space
next.

To convert plausibility into certainty, the language designer can use
a few tricks, such as asking the programmer, who knows more about the
context, to provide this assumption for them. It seems that the
programmer can make a lot of deductions from the nature of the
pointers they let inside the heap.

For instance, in the case of out-of-heap allocation for systems
programming (ancient heap, shared heap, arena...), this is a complete
non-issue even on 32-bit: one just needs to use the same standard
address space reservation technique, and never give back virtual
address space, so that it cannot overlap with any future assignments
for OCaml's heap. By courtesy, for symmetric reasons, we will avoid
giving back reserved space to the OS.

Also, the constraint 2. is trivially achieved in case one has 1TB of
address space reserved up-front and never need to reserve more. To
generalise this option, one can offer a runtime parameter where the
user can select a fixed virtual address space size if needed (e.g. for
32-bit).

We expect that the programmer might make further deductiond and take
further appropriate measures by studying the documentation of
whichever malloc they use (e.g. for the [glibc
malloc](https://sourceware.org/glibc/wiki/MallocInternals)).

Making 2. part of the norm further offers the possibility of detecting
such errors: when one sees a out-of-heap pointer during marking, one
can “taint” the corresponding virtual space, and fail on a future
address space reservation if one cannot obtain an untainted range.
Tainting is not exact (an address range can become reserved before an
out-of-heap pointer is seen by the GC), but serves to enforce 2. in
practice.

## Challenge 2: 32-bit support & large memory support

We aim to treat 32-bit and 64-bit similarly. With challenge 1 solved,
we can do similarly to Go and reserve the address space progressively.
This also addresses the problem in 64-bit for machines with more than
1TB of RAM, and offers the choice of virtual spaces smaller than 1TB
if needed.

We set a prefix size N ≥ 8. The runtime can reserve up to 2^N virtual
spaces, of size 2^L bytes (L=48-N in 64-bit and L=32-N in 32-bit),
aligned at 2^L. For 64-bit this means virtual spaces ≤1TB (for N = 16
this is 4GB, what the multicore GC reserves for the minor heap), and
≤16MB in 32-bit. We use a global 1-level or 2-level array of size (at
most) 2^N bytes as the address space map, so that finding the status
of the virtual space a pointer belongs to is done efficiently by
shifting and looking up in the array.

We consider 3 possible states for each space in the address space map:
Unknown, Reserved, Tainted. Thanks to the assumption 2. provided by
the programmer and the decision to never give back address space to
the OS, this state is monotonous: entries start with value Unknown and
may become Reserved or Tainted at some point, and no longer change.

When a major allocation requires to reserve additional contiguous
address spaces, such a reservation (aligned at 2^L, of size a multiple
of 2^L) is requested to the OS. A hint can be given as to where we
would like it to start. The given mapping is checked against the
address space map: none must be Tainted. In case of success, the
corresponding virtual spaces are marked as Reserved in the map, and in
case of failure then the rule 2. has been violated and the allocation
fails.

While not necessary to demonstrate our approach initially, many
real-world lessons can be learnt from the detailed sources of the Go
allocator. In Go, the size is chosen to be 64MB or 4MB depending on
the platform
[(details)](https://github.com/golang/go/blob/9acdc705e7d53a59b8418dd0beb570db35f7a744/src/runtime/malloc.go#L219-L231).
Given the small sizes, special care is taken to reserve space as
contiguously as possible to avoid fragmentation. In 64-bit, one hints
at a specific place in the middle of the address space unlikely to
conflict with other mappings
[(details)](https://github.com/golang/go/blob/9acdc705e7d53a59b8418dd0beb570db35f7a744/src/runtime/malloc.go#L489-L498).
In 32-bit one asks to start as low as possible and tries to reserve a
large chunk initially
[(details)](https://github.com/golang/go/blob/9acdc705e7d53a59b8418dd0beb570db35f7a744/src/runtime/malloc.go#L549-L564).
There are further details in
[malloc.go](https://github.com/golang/go/blob/master/src/runtime/malloc.go),
concerning more platforms than supported by OCaml.

## Challenge 3: synchronisation

As we explained, when a query to the address space map gives Unknown,
then we know we have an out-of-heap pointer, and we set it the entry
to Tainted. Thus, tainting virtual spaces has a further interesting
consequence: the value of the entry is permanently set to Reserved or
Tainted as early as after the first query against the address space
map. This gives a simple synchronisation strategy in multicore.

We give each domain a local map that caches the global map. Then,
following the same principle as before, each entry of the domain-local
map becomes permanently set to Reserved or Tainted after the first
query. If the first query yields Unknown in the local map, this time
one synchronises with the global map. If it is Unknown in the global
map, then it is permanently set to Tainted globally. Thus any
synchronisation when querying the map needs to happen at most once per
entry and per domain.

Furthermore, when extending the reserved space during an allocation,
one needs to be able to atomically compare and set several adjacent
entries. One then can use a mutex, since it only competes with (other
extensions of the reserved space and) queries of Unknown entries in
the global table, which happen at most 2^N times globally.

## Challenge 4: malloc for multicore

An interesting feature of multicore is that it relies on the system
allocator to manage large blocks. To avoid reimplementing a
high-performance multithreaded allocator, one can reuse what exists.
For instance, jemalloc arenas can be customised to use the memory
chunks one provides to it, e.g. carved out of the reserved address
space
([arena.i.extent_hooks](http://jemalloc.net/jemalloc.3.html#arena.i.extent_hooks)).

## Gathered evidence from the practice and the literature

### Interoperability

In his essay “Some were meant for C” (Onward! 2017,
https://www.cs.kent.ac.uk/people/staff/srk21/research/papers/kell17some-preprint.pdf),
Stephen Kell places the essential value of systems programming
languages in “communicativity” rather than performance. Among others,
he argues against “managed” languages in which “user code may deal
only with memory that the language implementation can account for”.

Concerning foreign pointers specifically, their allowance is indeed
standard in common (self-described) systems programming languages
(Rust, Go, Swift, and of course C/C++). For instance:

- Rust's ownership-and-borrowing features allows it to interoperate
  with garbage collected runtimes, and directly manipulate values from
  the foreign GCed heap (e.g. SpiderMonkey's GC, and Alan Jeffrey's
  “Josephine” https://arxiv.org/abs/1807.00067).

- In Rust, value interoperability with C++ is instrumental in the
  migration from one to the other because programs that are migrating
  can remain a long time in a hybrid state.

- https://github.com/apple/swift: “Swift is a high-performance system
  programming language. It has a clean and modern syntax, offers
  seamless access to existing C and Objective-C code and frameworks,
  and is memory safe by default." (see also
  https://www.uraimo.com/2016/04/07/swift-and-c-everything-you-need-to-know/)

There is ongoing research to augment OCaml with new kinds of types and
values which would allow operating more safely with foreign pointers
(among others): Radanne, Saffrich & Thiemann's
https://arxiv.org/abs/1908.09681, my own
https://arxiv.org/abs/1803.02796.

Proposed replacements in OCaml's “no-naked-pointers” mode include
introducing an indirection via a special block, or disguising the
pointer into an integer (using the lsb as a tag), i.e. as unstructured
data. One will be unable, for instance, to create an OCaml-like value
on the Rust stack or heap and pass it as an argument to an OCaml
function.

I am not aware of a common “systems” programming language that
requires such wrapping to access foreign objects. The discussion at
#9534, and the case of llvm bindings that has been reported on the
forums, shows preliminary empirical evidence of naked foreign pointers
in OCaml in the wild.

At #9534 and #9535, without questioning the no-naked-pointers
approach, library developers still reported the convenience of
allowing a special NULL pointer value for interoperability (and other
purposes). @xavierleroy commented: “we were hoping to get rid of those
code paths when naked pointers are disallowed, but with this PR the
special path is back, just to handle Obj.null.”

### Address space reservation

Reserving address space is standard in several common
garbage-collected languages: Java, GHC, and Go. GHC reserves 1TB by
default (https://gitlab.haskell.org/ghc/ghc/issues/9706). The
concurrent OCaml multicore GC reserves 4GB for the minor heap. The new
multi-threaded runtime for Julia reserves 100GB for thread stacks
(https://julialang.org/blog/2019/07/multithreading/). Go used to
reserve 512GB up-front at some point, and now reserves more
progressively.

Go further uses virtual addressing tricks to support interior
pointers: pointers to the inside of a block
(https://blog.golang.org/ismmkeynote). The bitpattern is used to
encode the size of the block and thus where it starts. (This technique
would have an obvious use in complement to the unboxing proposal for
OCaml, but is hard to adapt to a generational collector.)

Making use of large areas of reserved address space is a common
technique used in research about memory allocators. For instance,
“Archipelago: trading address space for reliability and security”
(ASPLOS 2008, https://emeryberger.com/research/archipelago/) reads:
“On modern architectures, especially 64-bit systems, virtual address
space is a plentiful resource.” In “Mesh: Compacting Memory Management
for C/C++ Applications”, a large virtual address space consumption is
traded for the ability to merge physical pages, in order to perform
“compaction without relocation” (PLDI 2019,
https://arxiv.org/abs/1902.04738).

(In fact, we are curious about adapting Mesh to work as the major
allocator, given that multicore does not implement compaction yet and
would otherwise need to stop the world for that. This would give the
ability to recognise in-heap pointers for free—but this is not a
universal solution given the possible performance overhead of Mesh for
people who do not need compaction, and its experimental nature.)

### Backwards compatibility

The transition to multicore requires to remove the page table.
Replacing the page table by a reserved virtual memory range offers the
possibility to avoid code breakage, and thus quicken the multicore
transition.

Indeed, backwards compatibility is a counter-intuitive topic. While
little documentation is found in the literature, Malloy and Powers
investigate the Python 2 vs. Python 3 split after ten years (resulting
from language changes that were expected to be minor) and conclude
that instead of favouring modernisation, the breaking changes in
Python 3 slowed the language evolution, with major software choosing
to confine themselves to “the diminishing intersection” of both
languages instead of using new features (ESE 2019,
https://link.springer.com/article/10.1007/s10664-018-9637-2). We do
not know how to predict the impact of breaking changes in programming
languages, even if they are believed to be minor.

It has been argued that code using foreign pointers could be adapted
in various ways:

- With an indirection: use custom or abstract blocks.
- Tag pointers: disguise pointers as integers by setting the lsb.

In addition to explicitly breaking existing code, as we have seen in
the case of foreign pointers, none of the propositions is ideal: in
theory both have a linear overhead when traversing a structure
compared to the same traversal using foreign pointers for instance, in
which case the code adaptation that is required could be
non-systematic. No empirical evaluation of the impact of the proposed
changes has been proposed yet (but an opam switch associated with
#9534 has recently appeared and will be a good tool to perform such an
evaluation).

In addition, the first solution incurs an allocation, whereas the
second solution only requires boilerplate, but confuses tools like
debuggers, static analysers and such, in ways that are unlikely to be
fixable over time. Moreover, they do not bring additional safety in
terms of resource-management: what they do is to deal in a rather
radical manner with the problem of growing the heap while remembering
who was outside, which is mostly required by the reliance on the
system malloc by the GC to request chunks of memory. This issue can
also be solved by virtual addressing techniques.

I asked @jberdine, who works at a large industrial OCaml user, if he
had any migration concerns. I quote his reply with his permission (I
preferred to keep it unaltered rather than quote it piecewise, to be
faithful to the nuanced view):

> [...] I have concerns about removing support for naked pointers, but
> just because it is not entirely clear how much effort it will take
> to port existing code to the new world. I am also unsure of the
> performance implications, but am hopeful that the net result will be
> fine.
>
> There are a few cases where I know that our code relies on libraries
> that use naked pointers (such as llvm) but I don’t know of a way to
> survey the transitive dependencies for code that relies on them. At
> the moment the best plan I have is to first rework the llvm
> bindings, and then build a compiler configured with
> no-naked-pointers and just debug the crashes. I have no idea how to
> estimate how much work that will be, which makes it a hard task to
> fit into other priorities. I wonder if there could be a way to
> identify the opam packages that are (transitively) no-naked-pointer
> compatible. [Note that this was written before #9534 appeared.]
>
> Another option might be to isolate the code that uses libraries that
> rely on naked pointers into separate executables which can be built
> with a naked-pointers compiler, and communicate via serialized
> values. But Marshaled values might not be compatible between
> naked-pointers and no-naked-pointers compilers, I’m not sure. This
> technique is probably only a transition stage though.
>
> I also need to figure out if e.g. ctypes or camlidl can be used to
> automate things like setting the low bit of a pointer when passing
> it from C to OCaml and clearing it on the path back.
>
> In my view, I agree that it is better to enable the multicore
> runtime to be faster than to make using naked pointers easy. It
> would be good if there was support in the tooling for interfacing
> between C and OCaml (e.g. the C FFI header files, ctypes, maybe
> camlidl) for the low-bit-setting technique of making naked pointers
> appear as integers. I expect that in the majority of cases, the
> naked pointers will be aligned, but as far as I understand there
> isn’t anything that guarantees this.

My comment: while a (non-precise) tool to identify naked pointers
proposed at #9534 partially solves one of the reported issue (and has
a cool implementation), other obstacles are mentioned, and a possible
"transition stage" involving a language split is mentioned. Yet it is
non-industrial users (academic, open source) who are usually the first
to lack the resources.

Lastly, another well-known use of out-of-heap pointers, which is known
to break with the no-naked-pointers mode, is symbolised by the Ancient
library and @gerdstolpmann's Netmulticore library
(http://blog.camlcity.org/blog/multicore1.html), which propose to
allocate OCaml values outside of the garbage-collected heap to offer
heaps that can be shared or large (such as larger than available RAM).
Neither an indirection nor tagged pointers can be used to replace this
use-case, a fact that has been known from the start
(https://sympa.inria.fr/sympa/arc/caml-list/2015-05/msg00088.html).

In addition, a third technique is sometimes mentioned, which is to
prefix the out-of-heap values with a black header. This technique is
unorthodox because it consists in checking whether one owns a pointer
by dereferencing it. One consequence is that such values can never be
deallocated, because it is never possible to reason whether they are
still reachable from the GC. For this reason, while this can be
appropriate for runtime-owned values that live for the duration of the
program, its interest for users is more marginal.

## Interest of out-of-heap allocation in current and future OCaml

I have mentioned that beyond foreign pointers, whose support already
benefits backwards-compatibility and interoperability, there are
libraries that allocate OCaml values outside of the heap. Here I
replace this usage in the context of evolving OCaml to make it a
better systems programming language.

### The thesis of complementary allocation methods

The Ancient and Netmulticore libraries perform out-of-heap allocation
for performance (large data, parallel computing). This usage of a
different memory management technique is better understood through the
lens of Bacon, Cheng and Rajan's classic paper “A unified theory of
garbage collection” (https://dl.acm.org/doi/10.1145/1035292.1028982),
which places memory reclamation techniques on a spectrum, with tracing
on one end (operating on live values) and reference-counting on the
other end (operating on dead values). Furthermore, the location on the
spectrum explains the advantages and drawbacks of each technique.

The use of out-of-heap allocation, not traversed by the GC and
explicitly freed when the programmer requests it, is on the opposite
end of the spectrum, and thus complements the OCaml GC with a
different set of advantages and drawbacks.

The emergence of safe and expressive abstractions to manage the latter
with _unique pointers_ in C++11 and Rust (which avoid the reference
count overhead, and prevent cycles that leak) has inspired a proposal
to develop out-of-heap allocation, in particular to improve the
capabilities of OCaml as a systems programming language (my “Resource
Polymorphism” proposal, https://arxiv.org/abs/1803.02796). Rust-like
ownership and borrowing would allow two techniques that are
complementary to the OCaml GC, as both benefit from a low latency and
the absence of repeated traversal by the GC for long-lived values:

- Fixed malloc-like allocation, highly tuned in modern allocators for
  small sizes. Compared to the generational GC, the user pays a bit
  more for allocation and release, but one also benefits from
  automatic cell re-use which statically eliminates some allocations.

- Arena (bump pointer) allocation, for values that tend to live longer
  than a minor collection but are deallocated all at once. Allocation
  and deallocation is very cheap, but comes with expressiveness
  restrictions.

Both would further benefit from a proposal to improve OCaml's control
on memory layout (https://github.com/ocaml/RFCs/pull/10), if
implemented.

### Low latency

A minor allocation risks incurring a stop roughly proportional to the
amount of promoted objects if it triggers a minor collection or a
major slice, frequently between 2ms and 10ms. The low latency of the
other allocation methods has the potential to drastically augment the
expressiveness of the special low-latency dialect of OCaml currently
in use, where one tries to promote very little.

In 1998, Mark Hayden's PhD thesis about the Ensemble system
(https://ecommons.cornell.edu/handle/1813/7316) found that
bump-pointer allocation using reusable off-heap arenas managed with
reference-counting was a solution to get (otherwise impossible)
reliable low latencies for buffer management in networking in OCaml. I
would like to make such usage first-class in OCaml.

Augmenting the low-latency capabilities of OCaml is one of the goals
to achieve better suitability as a systems programming language.

### Large heaps

The high GC CPU share on some allocation patterns with large data is
mentioned on the caml-list as a problem that has Ancient-like
allocation as a solution. This is a known problem with automatic
memory allocation studied in the context of so-called “Big data” where
allocations have an “epochal” behaviour (see for instance the
motivations in Nguyen et al., OSDI'16
https://www.usenix.org/system/files/conference/osdi16/osdi16-nguyen.pdf).

Furthermore, for data larger than RAM, fixed allocation is the only
solution to allocate in the swap, since the tracing GC is unsuitable
for operating on a disk: this is the original motivation of Ancient,
written in a time when RAM was more limited.

A recent experiment by @ppedrot with Coq to “hack a prototype on-disk
offloading similar to ocaml-ancient”, implemented by mashalling to a
file, saw a performance improvement of up to 12.3% in benchmarks
(µ=3.4%, σ=2.9)
(https://ci.inria.fr/coq/view/benchmarking/job/benchmark-part-of-the-branch/827/).

### Shared heaps

As far as shared memory between threads is concerned, the intent is
rather to benefit from the multicore GC, especially for the correct
reclamation of lock-free data structures, an area where Rust shows
limitations. Nevertheless, sharing memory between parallel threads is
a current application for off-heap allocation, for instance with
Netmulticore, that breaks under the no-naked-pointers mode.

@jberdine mentioned to me their shared heap which is rather large
(~100GB). I asked whether they would like such a heap to be scanned by
the GC in a possible reimplementation with multicore (quoting with his
permission again):

> Well, compacting it would be bad. But in at least some cases, we
> need what is essentially an off-heap GC for that huge shared memory.
> But yes, with such large heaps, it is very unclear what the
> performance profile will be, and whether the current off-heap shared
> memory systems can be replaced, or just augmented, by a parallel
> native heap.

They use marshalling rather than Ancient allocation, but the comment
should be relevant for large Netmulticore-style heaps as well.

### Non-moving heap

Fixed allocation is opposed to moving collectors. Not moving is better
suited for interoperability again: for instance, the lablgtk bindings
for GTK currently have to give the illusion that OCaml's GC is
non-moving, by calling `minor_collection` by hand and by disabling
compaction.

Not moving is also relevant for security, for instance in the case of
crypto APIs which `mlock()` a specific area in memory that contains
secrets. (In the particular case of secret keys and other unstructured
data, though, OCaml is already well-tooled thanks to bigarrays; but
the case of structured data has seen the development of other tools
such as cstruct.)

### Interoperability between GC and fixed allocation

In programming, a “no silver bullet” approach which seeks to offer
different tools (e.g. different memory allocation techniques) for
different jobs can only work if the tools work well in combination (as
opposed to leading to dialects that interact poorly: libraries that
cannot interoperate, etc.). In this section, I derive from this
requirement the key argument of why, even in a vacuum, a dynamic
“in-heap?” test is desirable.

When a data structure is allocated in the Ancient or Netmulticore
heaps, it can be manipulated like any other OCaml data structure.
Using simplified examples: a statically-allocated list can be
traversed using `List.filter` and the resulting list ends up correctly
GC-allocated; a large ancient-allocated tree can be traversed with a
zipper which uses the same principle as sharing to allocate only a
small portion of it with the GC during traversal. This degree of
interoperability, sharing and allocation polymorphism is not reachable
using the suggested alternatives: indirections or tagged pointers; nor
is it with other known alternatives: marshalling, or memory layouts
that would determine the allocation method statically à la “kinds as
calling conventions” (Eisenberg and Peyton Jones).

In “Resource Polymorphism”, I proposed a high-level interpretation of
this kind of interoperability between allocation methods, using an
ownership and borrowing model inspired from a categorical semantics.
In this model, the borrowing type operator (&) is interpreted as a
functor from a kind of "Ownership" types to a kind of "GCed" types,
that commutes with operations on types. In simple terms, this means
that one seeks to provide the same expressivity over borrowed values
as we know over common GC-allocated types, manipulate them with the
same functions, even if the borrowed value contains blocks allocated
outside of the heap (as in the example of the ancient-allocated zipper
mentioned above, that mixes blocks allocated with the GC and blocks
allocated outside of the heap). The existence of such an operator
depends on the existence of types of borrowed values that are agnostic
regarding the allocation methods of each kind, and thus for which a
_dynamic_ “in-heap?” check is a requirement.

While this is some of the most forward-looking part of the
resource-management proposal, this matches what can already be done
with ancient-like allocation in a wild and unsafe manner, and thus it
simply asks in the meanwhile that we do not break currently-working
code.

Other works from researchers in the OCaml community relate to these
goals. Similar goals are given in Gabriel Scherer's research proposal
for a “Low-level Ocaml”
(http://gallium.inria.fr/~scherer/topics/low-level-ocaml.pdf). I
already mentioned Radanne, Saffrich & Thiemann
(https://arxiv.org/abs/1908.09681), which explores some of the
complementary machinery necessary to make dynamically-allocated memory
safe.

Of course, these investigations, prospective by nature, are beyond the
scope of this RFC. But they serve to demonstrate the community's
motivation in pursuing these goals.

## Dispelling common misconceptions

While virtual address space reservation is standard, I have learnt
many misconceptions about it. Here I list and try to dispell some of
them, in the hope of focusing the conversation on actual drawbacks and
design requirements. More knowledgeable people are welcome to correct
me. (Please be gentle—this is knowledge outside of my main area of
expertise, which I had to develop because I seemed to be facing vague
claims on the topic.)

The misconceptions are probably not helped by the fact that the first
implementations of OpenJDK and Golang under Linux were done
incorrectly and suffered from real drawbacks. Let us recall that under
Linux one essentially reserves virtual address space with `mmap` with
flag `PROT_NONE`, and one commits portions of this space using
`mprotect` with flags `PROT_READ|PROT_WRITE`. Possible mistakes
involve the reliance on overcommitting together with `MAP_NORESERVE`,
instead of address space reservation, among others. Windows is more
straightforward with calls to `VirtualAlloc` with options
`MEM_RESERVE` and `MEM_COMMIT`.

This technique was probably not as well documented and crowd-sourced
until recent years, and for similar reasons probably, we found aspects
which could be improved in the prototype from 7 years ago at
ocaml/ocaml#6101 (superfluous use of MAP_NORESERVE, usage of `mmap`
instead of `mprotect`). (The implementation from OCaml multicore,
however, seems to have been done in the right way from the beginning.)

### One cannot use setrlimit with RLIMIT_AS to limit memory

While this one is correct, this is a limitation of setrlimit which
only offers to limit virtual memory, and not the resident set. The
multicore concurrent collector, with the 4GB reserved space for minor
heaps, already considered to give up on it. Since Linux 4.7,
allocation done with mmap are counted for the RLIMIT_DATA resource
limit (see `malloc(3)`). In addition, more advanced memory limits are
provided by cgroups, and according to documentation they would work
correctly.

### One cannot use `mlock` to avoid swapping

One can, use the flag `MCL_ONFAULT`.

### This forces to use overcommitting

No, use `PROT_NONE`.

### I will have OOM due to `vm.overcommit_ratio`

No, see previous point.

### I risk having a SIGSEGV the first time I write to a page

You do not need the `MAP_NORESERVE` mmap flag, whether you have
overcommitting enabled or not.

### The core dumps are 1TB

They are stored as sparse files, or you forgot `PROT_NONE` and
inadvertently rely on overcommitting. (But gdb's own core dump may
need further help from `madvise(MADV_DONTDUMP)` and
`madvise(MADV_DODUMP)`, or at least it did at some point.)

### Reclaiming mmaped memory is expensive

The problem with TLB shootdowns when reclaiming memory under Linux
seem to concern all allocators, but at least by controlling the
mapping we can ensure that reclamation only happens during calls to
`Gc.compact()` or another appropriate moment if we want. The current
allocator design for multicore defers to the system allocator for
large allocations, and as far as I understand does not offer similar
control.

## Thanks

Thanks to Josh Berdine, Frédéric Bour, Rian Douglas, Jacques-Henri
Jourdan, Adrien Guatto, and Gabriel Scherer for discussions on this
topic.
