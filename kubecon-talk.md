# kubecon-talk 30 minutes
## INTRO
Hey. I'm Eirik aka clux on github and am one of the main maintainers on kube-rs.

Today, talking about the kubernetes api, some of the generic assumptions and invariants that kubernetes wants to maintain, but for the lack of actual generics in the language, is either a best-effort ordeal, and in other broken.

We'll talk a little bit about how a richer type system - like rust's - gives us more a lot more for free in this regard, but with the caveat that we are still building on top of kubernetes' api, which is written in go.
=> Broken invariants need to be respected in rust land as well.

But this is still meant to be a pretty positive talk. Yes, some invariants are broken, but regardless, kubernetes is still remarkably consistent in its api despite shortcomings of the language.

Additionally, this might serve as a bit of a high level view into async rust (which was released on stable in just about a year ago - so there's been tons of advancements there). IMO, it's now in a really good place, library ecosystem is great and starting to properly stabilize. However, the learning curve is ever present. And there are some rough edges.

## Kubernetes
Let's talk about kubernetes provides.

## THE GOOD PARTS
### apimachinery types.go
So let's dive into the most important file of all. Meta types in apimachinery.

TypeMeta.
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L41-L56
Every object has kind + version - flattened into the root structure like `Pod`

ObjectMeta.
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L110-L282
Every object MUST Have metadata, and must look like this. There's OwnerReferences, labels, annotations, and finalizers that all can go in there, and they're standardised. Every object supports these.

List types.
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L914-L923
For when you ask for a collection of items. And look at items there; a dynamic collection so this struct can be re-used.

APIResource.
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L999-L1032
standardising where we we can get information of what Kind

ListOptions
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L329-L346
GetOptions, ListOptions, DeleteOptions, PatchOptions. All parameters that the API accepts encapsulated into common structs from this root file. Error responses.

LabelSelectors.
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L1095-L1104
that sits inside ListOptions, so there's a generic way of filtering

Sorry to go on for so long about this file, but the consistency and complete adoption of this file is they key to what makes any generics in other languages possible.

### client-go consistency
The same consistency can be seen in client-go
look at the interface to Deployment (say)
https://github.com/kubernetes/client-go/blob/master/kubernetes/typed/apps/v1/deployment.go#L41-L55
getters/updaters/patchers/replacers/listers/deleters/watchers
they take the same parameters

and you can go to any other type and you'll see the same story.
this is a 200 line file for deployment. there's one for every object?
how could be this possibly be consistent?
https://github.com/kubernetes/client-go/blob/master/kubernetes/typed/apps/v1/deployment.go#L17

right. all of this is generated.
because people recognised you have to enforce some of these assumptions for them to stick.

now, this isn't generics, but it's consistency.
for each, kind, the specific structs are specialized manually
via code generation - but the source is present regardless

and it's not the only file generated.
informers logic for every type is also there
https://github.com/kubernetes/client-go/blob/master/informers/apps/v1/statefulset.go#L58-L78

as a result; client-go > 100K LOC (without vendoring)

and i'm not trying to judge here. this is great.
the fact that everything looks the same is what enables `kubectl` to provide such a consistent interface.

### api endpoints
url consistency lets us make easy mappings between types and urls

**Cluster-scoped resources**
```
GET /apis/GROUP/VERSION/RESOURCETYPE
GET /apis/GROUP/VERSION/RESOURCETYPE/NAME
```
**Namespace-scoped resources**
```
GET /apis/GROUP/VERSION/RESOURCETYPE
GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE
GET /apis/GROUP/VERSION/namespaces/NAMESPACE/RESOURCETYPE/NAME
```

https://kubernetes.io/docs/reference/using-api/api-concepts/#standard-api-terminology

though things start to break down a little bit here even though this is straight out of the "Standard API Terminology" page on the kubernetes website.
because this does not hold for pods, nodes, namespaces, and any other type in the core object list. they have a different url that starts with `api` rather than `apis`.

```
GET /api/v1/pods
```

but ok that's fine, we can strip a slash if the group is empty and then change change apis to api...K.

## WatchEvents
WatchEvents are what you received when you perform a watch call, aka a GET on a root resource api. With watch parameters in the querystring. When you use watch, you effectively set a timeout, and you'll get a chunked response, of NEWLINE delimited json, each line containg a wrapped verision of your object

```
{ "type": "ADDED", "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...} } }
{ "type": "MODIFIED", "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...} }
```

so for each line you can parse the inner object as the type you actually have.
oh, and since these objects are frequently bigger than the MTU, any client would need to buffer chunks until you have a complete line.

so we can work with that. all apis use this and it's consistent.

well.. at least it was. so let's move on to broken assumptions.

## BROKEN ASSUMPTIONS
### Object<Spec, Status>
What people tell you it's like. Bring up some snowflakes.

### Optional everything
even though a resource having a name inside a namespace is a fundamental idea

metadata.name optional (yes, because of `generatename`..)
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L117-L118
so there's a decicion that now causes all clients to have to deref
rather than distinguish between partial data accepted as input and finalized stored input

### Optional metadata
screenshot code with the +optional... in pod?
https://github.com/kubernetes/api/blob/master/core/v1/types.go#L3667-L3686

### empty api group
in general we have url consistency, but not for core types (in empty group)

### conditions..
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L1367

### watch is broken
mention many issues, stale rvs, relisting required from a client re-watch every <300s. so much data (node informer, hah). can't filter out events.
writing controller to reconcile? you'll trigger your own loop.. (TODO: verify)

### watchevent is weird for bookmarks
does not pack object inside

## WHERE TO DRAW THE LINE
Show where we are. Evolving target.

## THANKS
first a few thanks.. I'll be talking about a grab bag of different things, but from the perspective of [kube-rs](https://github.com/clux/kube-rs/).

- Arnav Singh / @Arnavion for k8s-openapi
generates structures from openapi schemas, as well as factoring out several traits that is then implemented for these structures
the project really is the lynchpin that makes any generics possible

### Metadata
From k8s-metadata. Trait with associated constants. Codegen fills this in. Lynchpin.
Type system here is effectively telling you that these constants are available for every struct that implements this trait. And every k8s-openapi type implements it. So you just have to import the trait to be able to read these values.

### Resource
Show the core props + what we need to use api. Params objects. One FILE!
CAVEAT: load-bearing pluralize.
phrase i had never believed i had to use to describe software architecture, let alone from my own designs, but here we are.

## Types.go linkin
Remember when I mentioned all the structs in types.go? These are the ones we define in kube-rs.
## List types
We also have ListParams, PatchParams.

## Dynamic API
Show resource.rs converting into bytestream.
Of course, this isn't really what we want. We don't want to be interjecting at every point of the way to try to deserialize a bytestream into a concrete type.

### Api<K> trait where K: Metadata
Show how to generate all those methods you saw in client-go across all types with a blanket impl.

### In general: Lean on types
trying to catch errors with type safety rather than --pattern and passive code generation (like kubebuilder)

## Code Generation
## #[derive(Serialize, Deserialize)]
## #[serde(rename_all = "camelCase")]
## #[derive(CustomResource)]
## #[kube(group = "clux.dev", version = "v1", namespaced)]

So we can do all the necessary code generation that doesn't completely fit within a strict typesystem with procedural macros. They are effectively a way to generate code, but it's a first class citizen of cargo; rust's build system and package manager.

When you `cargo build`, these procedural macros generate code which is then used in the main compilation stage. So that whole class of errors where you are operating on a stale version of generated code, can just disappear.

### Watch
Mention hard parts briefly. Chunking. Async. impl Stream == async iterator.
..but re-list

## Runtime
How to build on top of watch and the api?

- Teo K. Röijezon / @teozkr for kube-runtime
Figured out an entirely Stream based solution for reflectors/watchers and controllers, and rewrote the entire runtime part of `kube`. It's an amazing techncial achievement that we're just barely starting to gain the benefit of.

### Watcher
Informer-like. But FSM.

### Reflector
Builds on top of watcher and adds a store. Move ensures no use after construction. Writer disappears. No weird contracts in godoc. Enforce it in the code.

### Controller
The big one...


## Building Controllers
not rehashing best practices. most advice from kubebuilder / controller-runtime applies. reconcile needs to be idempotent, check state of the world before you redo all the work on a duplicate event. use server side apply.

async/streams/tokio/web frameworks/metrics/tracing that makes writing controllers in rust very enjoyable. THOUGH WITH CAVEAT;

## Examples
Mention streams need to be polled.
Mention boxing.

## Caveats
Rough edges. Testing story (can be done now with streams).
