# kubecon-talk
## INTRO
Hey. I'm Eirik aka clux on github and am one of the main maintainers on kube-rs.

Today, talking about the kubernetes api, generic assumptions and invariants that kubernetes wants to maintain, but is for the lack of actual generics in the language, is in many cases a best-effort ordeal, and in other cases completely broken.

We'll talk a little bit about how a richer type system like rust gives us more a lot more for free in this regard, but with the caveat that we are still building on top of kubernetes' api, which is written in go.
=> Broken invariants need to be respected in rust land as well.

But this is still meant to be a pretty positive talk. Yes, some invariants are broken, but kubernetes is still remarkably consistent in its api despite shortcomings of the language.

Also, I hope that you'll get a little bit of a high level overview of some of the recent advancements in rust like; async/streams/tokio/web frameworks/metrics/tracing that makes writing controllers in rust very enjoyable. THOUGH WITH CAVEAT; many rough edges atm.

## THANKS
- Arnav Singh / @Arnavion for k8s-openapi
provides structures from openapi schemas, as well as factoring out
the project really is the lynchpin that makes any generics possible

- Teo K. Röijezon / @teozkr for kube-runtime
Figured out an entirely Stream based solution for reflectors/watchers and controllers. It's an amazing techncial achievement that we're just barely starting to gain the benefit of.

Will talk a little more about these two as we go on.

But also wanna thank

- .... TODO:...

for making kube-rs be a viable option and making it work in tons of configurations.

## THE GOOD PARTS
### apimachinery types.go
MetaData. ObjectMeta. All the stuff in apimachinery
### api endpoints
url consistency lets us make easy mappings between types and urls

### client-go consistency
methods the same on most types
getters/updaters/patchers/replacers/listers/deleters
they take the same parameters everywhere
watch possible on everything
WatchEvent packs the object itself inside

## BROKEN ASSUMPTIONS
### Object<Spec, Status>
What people tell you it's like. Bring up some snowflakes.

### Optional everything
even though a resource having a name inside a namespace is a fundamental idea
metadata.name optional (yes, because that name initializer..)

### Optional metadata
screenshot code with the +optional... in pod?

### empty api group
in general we have url consistency, but not for core types (in empty group)

### watch is broken
mention many issues, stale rvs, relisting required from a client re-watch every <300s. so much data (node informer, hah). can't filter out events.
writing controller to reconcile? you'll trigger your own loop.. (TODO: verify)

### watchevent is weird for bookmarks
does not pack object inside

## WHERE TO DRAW THE LINE
Show where we are. Evolving target.

### Metadata
From k8s-metadata. Trait with associated constants. Codegen fills this in. Lynchpin.

### Resource
Show the core props + what we need to use api. Params objects. One FILE!
CAVEAT: load-bearing pluralize.

### Api<K> trait where K: Metadata
Show how to generate all those methods you saw in client-go across all types with a blanket impl.

### In general: Lean on types
trying to catch errors with type safety rather than --pattern and passive code generation (like kubebuilder)

### Watch
Mention hard parts briefly. Chunking. Async. impl Stream == async iterator.
..but re-list

## Runtime
How to build on top of watch and the api?

### Watcher
Informer-like. But FSM.

### Reflector
Builds on top of watcher and adds a store. Move ensures no use after construction. Writer disappears. No weird contracts in godoc. Enforce it in the code.

### Controller
The big one...


## Building Controllers
not rehashing best practices. most advice from kubebuilder / controller-runtime applies. reconcile needs to be idempotent, check state of the world before you redo all the work on a duplicate event. use server side apply.

## Examples
Mention streams need to be polled.
Mention boxing.

## Caveats
Rough edges. Testing story (can be done now with streams).
