# kubecon-talk 30 minutes
## INTRO
Hey. I'm Eirik aka clux on github and am one of the main maintainers on kube-rs.

Today, talking about the kubernetes api, some of the generic assumptions and invariants that kubernetes wants to maintain, but for the lack of actual generics in the language, _the properties_ are generally enforced through consistency and manual code-generation steps.

We'll talk a little bit about how rust, with its richer type system, gives us the same consistency for free, and lets us model the api easily. Still, it's not a magic bullet. Any broken invariants on the Go side would still need to be respected in rust land.

But in the mean time; this is still going to be a very positive talk. Yes, there are some broken invariants, but regardless, kubernetes is remarkably consistent in its api despite shortcomings of the language. And we'll show some examples of this from source.

We'll also touch on a bit of async api design in rust during the process of modelling the api with rust generics. Async rust was only properly released about a year ago, and the rust ecosystem has consequently seen enormous advances in this year. So if you're not up to speed, you'll at least see some patterns in this talk.

## Kubernetes
Let's talk about what kubernetes provides.

### meta types.go in apimachinery
So let's dive into the most important file of all.

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

So I am raving this about this, but it's because of the consistency and complete adoption of everything in this; that kubernetes feels so consistent and why we can actually make generic assumptions in other languages.

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
because people recognised that you **have** to enforce some of these assumptions for them to stick.

now, this isn't generics, but it's consistency.
for each, kind, the specific structs are specialized manually
via code generation - but the source is present in repo regardless

and it's not the only file generated.
informers logic for every type is also there
https://github.com/kubernetes/client-go/blob/master/informers/apps/v1/statefulset.go#L58-L78

as a result; client-go > 100K LOC (without vendoring)

and again i'm not trying to judge here. this is great.
the fact that everything looks the same in here, is what enables `kubectl` to provide such a consistent interface.

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

### Broken: empty api group
because this does not hold for pods, nodes, namespaces (TODO: more ex), and any other type in the core object list. they have a different url that starts with `api` rather than `apis`.

```
GET /api/v1/pods
```

but that's a relatively minor inconsistency, we can strip a slash if the group is empty and then change change apis to api...K.

## WatchEvents
WatchEvents are what you received when you perform a watch call, aka a GET on a root resource api. With watch parameters in the querystring. When you use watch, you effectively set a timeout, and you'll get a chunked response, of NEWLINE delimited json, each line containg a wrapped verision of your object

```
{ "type": "ADDED", "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "10596", ...} } }
{ "type": "MODIFIED", "object": {"kind": "Pod", "apiVersion": "v1", "metadata": {"resourceVersion": "11020", ...}, ...} }
```

so for each line you can parse the inner object as the type you actually have.
oh, and since these objects are frequently bigger than the MTU, any client would need to buffer chunks until you have a complete line.

so we can work with that. all apis use this and it's consistent.

## END PRAISE - CONSTRUCT AROUND IN RUST

## THANKS
first a few thanks.. I'll be talking about a grab bag of different things, but from the perspective of [kube-rs](https://github.com/clux/kube-rs/).

- Arnav Singh / @Arnavion for `k8s-openapi`
generates rust structures from openapi schemas, plus as factoring out several traits that is then implemented for these structures
the project really is the lynchpin that makes any generics possible

### Resource Trait
From `k8s-openapi`. Type system here is effectively telling you that these constants are available for every struct that implements this trait. So you just have to import the trait to be able to read these values.

Arnav's Codegen implements this trait for every kubernetes object

```rust
pub trait Resource {
    const API_VERSION: &'static str;
    const GROUP: &'static str;
    const KIND: &'static str;
    const VERSION: &'static str;
}
```

Normally traits are meant to encapsulate behaviour, but you are allowed to put in static associated constants. 

### Metadata Trait
Another one from `k8s-openapi`.

```rust
pub trait Metadata: Resource {
    type MetaType;
    fn metadata(&self) -> &Self::MetaType;
}
```
TODO: simple first, then Oslight complication due to metatype. TODO: objectmeta only?

Tells you that every object that implements it, has a way to extract a reference to its metadata. Can configure what the metadata type `Ty` actually is, but in 99% of cases it's `ObjectMeta`, and the other is `List<T>` which uses `ListMeta`.

### Two root traits - what can we do
Let's try something naive first.

#### Broken: Object<Spec, Status>
TODO: rephrase
Who's heard this. A k8s object consists only of `apiVersion` + `kind`, `metadata`, `spec`, `status` structs? What people tell you it's like. Even maintainers will use this simplification.

```rust
pub struct Object<Spec, Status> {
    pub types: TypeMeta, // apiVersion + kind
    pub metadata: ObjectMeta,
    pub spec: Spec,
    pub status: Option<Status>,
}
```

how this would look in rust. (NB: Omitting some details). Notice we can actually model this very easily. `Spec` and `Status` here are generic types and are specialized at compile time for the various invocations.

The problem with this is that it's not in general true.
In case you've forgottene about how these look, or used just crds for so long. Here's an awkward reminder of snowflake objects.

### Broken: Snowflakes
Look at configmap (data +  binary_data). Fields at the top level.

```rust
pub struct ConfigMap {
    pub metadata: ObjectMeta,
    pub binary_data: Option<BTreeMap<String, ByteString>>,
    pub data: Option<BTreeMap<String, String>>,
    pub immutable: Option<bool>,
}
```

similar story for secret:

```rust
pub struct Secret {
    pub metadata: ObjectMeta,
    pub data: Option<BTreeMap<String, ByteString>>,
    pub immutable: Option<bool>,
    pub string_data: Option<BTreeMap<String, String>>,
    pub type_: Option<String>,
}
```

```rust
pub struct ServiceAccount {
    pub metadata: ObjectMeta,
    pub automount_service_account_token: Option<bool>,
    pub image_pull_secrets: Option<Vec<LocalObjectReference>>,
    pub secrets: Option<Vec<ObjectReference>>,
}
```
(no spec, automount bool)

tons more `Endpoint` (subsets vec), `Role` (rules obj), `RoleBinding` (subjects + roleRef).

and the wtf struct `Event`, with 15 random fields:

```rust
pub struct Event {
    pub metadata: ObjectMeta,
    pub action: Option<String>,
    pub count: Option<i32>,
    pub event_time: Option<MicroTime>,
    pub first_timestamp: Option<Time>,
    pub involved_object: ObjectReference,
    pub last_timestamp: Option<Time>,
    pub message: Option<String>,
    pub reason: Option<String>,
    pub related: Option<ObjectReference>,
    pub reporting_component: Option<String>,
    pub reporting_instance: Option<String>,
    pub series: Option<EventSeries>,
    pub source: Option<EventSource>,
    pub type_: Option<String>,
}
```

The core objects really cause a lot of trouble. Can't rely on SPEC/STATUS (TODO: gets us into trouble for api later).


=> if we can't rely on spec/status, what about metadata props?

#### Broken: Optional metadata
screenshot code with the +optional... in pod?
https://github.com/kubernetes/api/blob/master/core/v1/types.go#L3667-L3686
...how? we said we had that guarantee?

we think this is because `patch` requests that allow sending empty metadata in the body (only acts on spec/status (dep on what you patch), name already inferrable from the url).

so this is one we deliberately disobey.
because it makes it so awkward to unwrap something that has to be there (except in weird manual stuff you write yourself)

but in general, have to obey all optionals...:

#### Broken: Optional names
even though a resource having a name inside a namespace is a fundamental idea

metadata.name optional (`generatename` mechanism)
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L117-L118
makes sense, but now every clients now have to assume non-null
it's an easy assumption to make, but it's a prominent example of many
and it leads you down a very uneasy road having to unwrap every option


### Resource
Show `Resource`. Note basically a copy of data that's in the `Resource` trait + the dynamic property of what namespace it lives in (if it's a namespaced resource).

```rust
#[derive(Clone, Debug)]
pub struct Resource {
    pub api_version: String,
    pub group: String,
    pub kind: String,
    pub version: String,
    pub namespace: Option<String>,
}
```

can create manually, or instant win ctor (assuming )

```rust
impl Resource {
    pub fn namespaced<K: k8s_openapi::Resource>(ns: &str) -> Self {
        Self {
            api_version: K::API_VERSION.to_string(),
            kind: K::KIND.to_string(),
            group: K::GROUP.to_string(),
            version: K::VERSION.to_string(),
            namespace: Some(ns.to_string()),
        }
    }
}
```

can also define the function that dictates all of k8s urls:

```rust
impl Resource {
    fn make_url(&self) -> String {
        let ns = self.namespace.as_ref().map(|n| format!("namespaces/{}/", n));
        format!(
            "/{group}/{api_version}/{namespaces}{resource}",
            group = if self.group.is_empty() { "api" } else { "apis" },
            api_version = self.api_version,
            namespaces = ns.unwrap_or_default(),
            resource = to_plural(&self.kind.to_ascii_lowercase()),
        )
    }
}
```

CAVEAT: load-bearing pluralize.
phrase i had never believed i had to use to describe software architecture, let alone from my own designs, but here we are.

### Urls => PatchParams/CreateParams (types.go)
Remember when I mentioned all the structs in types.go? These are some of thne few structs we define in kube-rs. Can be used to create:

## Dynamic API
Show resource.rs converting into bytestream.
Of course, this isn't really what we want. We don't want to be interjecting at every point of the way to try to deserialize a bytestream into a concrete type.

```rust
impl Resource {
    pub fn create(&self, pp: &PostParams, data: Vec<u8>) -> Result<Request<Vec<u8>>> {
        let base_url = self.make_url() + "?";
        let mut qp = url::form_urlencoded::Serializer::new(base_url);
        if pp.dry_run {
            qp.append_pair("dryRun", "All");
        }
        let urlstr = qp.finish();
        let req = http::Request::post(urlstr);
        req.body(data).map_err(Error::HttpError)
    }
}
```

### Api<K> where K: Metadata
Show how to generate all those methods you saw in client-go across all types with a blanket impl.

```rust
impl<K> Api<K>
where K: Clone + Deserialize + Metadata,
{
    pub async fn create(&self, pp: &PostParams, data: &K) -> Result<K>
    where K: Serialize,
    {
        let bytes = serde_json::to_vec(&data)?;
        let req = self.resource.create(&pp, bytes)?;
        self.client.request::<K>(req).await
    }
}
```

### In general: Lean on types
trying to catch errors with type safety rather than --pattern and passive code generation (like kubebuilder)

## Code Generation
### Serialize
```rust
#[derive(Serialize, Deserialize)]
pub struct MyFoo {
    name: String,
    info: Option<String>,
}
```
tons of extra things serde can do, similar to go `serde(rename_all = "camelCase")`

So we can do all the necessary code generation that doesn't completely fit within a strict typesystem with procedural macros. They are effectively a way to generate code, but it's a first class citizen of cargo; rust's build system and package manager.

When you `cargo build`, these procedural macros generate code which is then used in the main compilation stage. So that whole class of errors where you are operating on a stale version of generated code, can just disappear.

## CustomResource
```rust
#[derive(CustomResource, Serialize, Deserialize, Clone)]
#[kube(group = "clux.dev", version = "v1", kind = "Foo", namespaced)]
#[kube(status = "FooStatus")
pub struct MyFoo {
    name: String,
    info: Option<String>,
}
```

### SKIP: Broken: Conditions
while we are talking about conditions
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L1367
sits inside a vector, so they all look the same, so you have to always filter the conditions for the type you want. presumably so that `kubectl describe` can display all conditions in one nice table.

but, we could deal with that. What we cannot deal with is that you cannot really `patch_status` to update particular condition entry.

none of the original patch types even work (strategic might have, but not supported on crds). so you need at least server side apply to even use conditions.

SKIP DUE TO https://github.com/clux/kube-rs/issues/43 FIXED IN SS APPLY?

### Watch
Mention hard parts briefly. Chunking. Async. impl Stream == async iterator.
..but re-list

### Broken: Watch
mention many issues, stale rvs, relisting required from a client re-watch every <300s.

then the amount of data. tried using a node informer? sooo much noise. FULL 10k data every 5s because the conditions in its status object contain a last updated timestamp...

### WatchEvent
Then WatchEvent itself. Remember how watch events all packed an object inside of it? We can model this in rust with generic enums:

```rust
#[derive(Deserialize, Serialize, Clone)]
#[serde(tag = "type", content = "object", rename_all = "UPPERCASE")]
pub enum WatchEvent<K> {
    Added(K),
    Modified(K),
    Deleted(K),
    Bookmark(K),
    Error(ErrorResponse),
}
```

The serde tags here tells serde that the values of the enum variants are put inside on the object key, and the enum variant name on a key call tag (which are sent/recvd as uppercase - to match go convention). so this is actually really nice.

that's how that would look. however, this is one of those small cases where kubernetes actually pulls out all the optionals.

```json
{"type":"BOOKMARK","object":{"kind":"Pod","apiVersion":"v1","metadata":{"resourceVersion":"3845","creationTimestamp":null},"spec":{"containers":null},"status":{}}}
```

no spec, no name, kind Pod.
so that actually validates `metadata.name` being optional (even if we didn't have a generatename mechanism).

```rust
#[derive(Deserialize, Serialize, Clone)]
#[serde(tag = "type", content = "object", rename_all = "UPPERCASE")]
pub enum WatchEvent<K> {
    Added(K),
    Modified(K),
    Deleted(K),
    Bookmark(Bookmark),
    Error(ErrorResponse),
}
```

## Runtime
How to build on top of watch and the api. Well we got to watch continously, but not longer than 5 minutes, propagate all user errors, retry/re-list on desync errors, and still somehow encapsulate it all in one nice stream. It's absolutely not trivial.

So a huge shoutout to my other maintainer:

- Teo K. Röijezon / @teozkr who wrote kube-runtime (controller-runtime equivalent)

He basically figured out an entirely Stream based solution for watchers/reflectors and controllers, and rewrote that entire module of `kube`.

It's an amazing technical achievement that makes it really easy to integrate into your application.

### Watcher
Informer-like. But FSM.

```rust
enum State<K: Meta + Clone> {
    /// Empty state, awaiting a LIST
    Empty,
    /// LIST complete, can start watching
    InitListed { resource_version: String },
    /// Watching, can awaited stream (But on desync, move back to Empty)
    Watching {
        resource_version: String,
        stream: BoxStream<'static, Result<WatchEvent<K>>>,
    },
}
```

the last magic there is just "a stream of WatchEvent results of type K", put inside a box on the heap.

### Reflector
Builds on top of watcher and adds a store.

```rust
let cms: Api<ConfigMap> = Api::namespaced(client, &namespace);

let store = reflector::store::Writer::<ConfigMap>::default();
let reader = store.as_reader();
let rf = reflector(store, watcher(cms, lp));
```

Move ensures no use after construction. Writer disappears. No weird contracts in godoc. Enforce it in the code.

what is a reflector?

```rust
pub fn reflector<K: Meta + Clone, W: Stream<Item = Result<watcher::Event<K>>>>(
    mut store: store::Writer<K>,
    stream: W,
) -> impl Stream<Item = W::Item> {
    stream.inspect_ok(move |event| store.apply_watcher_event(event))
}
```

### Controller
Controller is a system that calls your reconciler with events as configured.
You define 2 fns. One where you write idempotent (not going to talk about how to write resilient controllers, all normal advice (kbuilder etc) applies).
Second one is an error handler. You might want to check every error dilligently within the reconciler, but you can also just use `?`.

```rust
async fn reconcile(g: ConfigMapGenerator, ctx: Context<()>) -> Result<ReconcilerAction, Error> {
    // TODO: reconcile
    Ok(ReconcilerAction {
        requeue_after: Some(Duration::from_secs(300)),
    })
}
fn error_policy(_error: &Error, ctx: Context<()>) -> ReconcilerAction {
    // TODO: handle non-Oks from reconcile
    ReconcilerAction {
        requeue_after: Some(Duration::from_secs(60)),
    }
}
```

if you have those, then it's just hooking up events and contexts:

```rust
async fn main() -> Result<(), kube::Error> {
    let client = Client::try_default().await?;
    let context = Context::new(()); // bad empty context - put client in here
    let cmgs = Api::<ConfigMapGenerator>::all(client.clone());
    let cms = Api::<ConfigMap>::all(client.clone());
    Controller::new(cmgs, ListParams::default())
        .owns(cms, ListParams::default())
        .run(reconcile, error_policy, context)
        .await;
    Ok(())
}
```

should remind you a bit of controller-runtime. heavily inspired (got help).

## Building Controllers
not rehashing best practices. most advice from kubebuilder / controller-runtime applies. reconcile needs to be idempotent, check state of the world before you redo all the work on a duplicate event. use server side apply. use finalizers to gc.

## Examples
No scaffolding here. Choose your own dependencies.
Web frameworks?
- actix
- warp
- rocket

metrics libraries, logging libraries, tracing libraries,
- prometheus
- tracing (#[instrument] -> spans! (part of tokio))
- (tracing has log exporters, so just start with tracing, want jaeger anyway)
- sentry

ultimately, not going to dictate anything and put it inside an opinionated framework.

link to controller-rs and version-rs.

## Caveats
Rough edges. Api library (kube) quite stable, but kube-runtime is pretty new still.
