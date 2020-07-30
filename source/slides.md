### Hidden Generics in Kubernetes' API
<style type="text/css">
  .reveal h3, .reveal p, .reveal h4 {
    text-transform: none;
    text-align: left;
  }
  .reveal ul {
    display: block;
  }
  .reveal ol {
    display: block;
  }
  .reveal {
    background: #353535 !important;
  }
</style>

- Eirik Albrigtsen
- [clux](https://github.com/clux) / [@sszynrae](https://twitter.com/sszynrae)
- [kube-rs](https://github.com/clux/kube-rs)

notes:
- eirik/clux - one of the main maintainers on kube-rs.
- talking about the kubernetes api, some of the generic assumptions and invariants that kubernetes wants to maintain, but for the lack of actual generics in the language, _these invariants are generally enforced through consistency and code-generation steps.

---
### Hidden Generics in Kubernetes' API

- Finding invarints in Go codebase
- Use Rust Generics to model the API <!-- .element: class="fragment" -->
- Async Rust <!-- .element: class="fragment" -->

notes:
- We'll identify some of these invariants while covering parts the kubernetes codebase.
- Then talk about how to model the same api in rust using generics, and see that it gives us the same consistency more-or-less for free.
- We'll also touch on async api design in rust during this modelling process. Since async rust was only properly released about a year ago, so if you're not familiar, you'll at least see some of the now more established patterns.


<!--Still, it's not a magic bullet. Kubernetes is written in Go; Any broken invariants on the Go side would still need to be respected in rust land.
Yes, there are some broken invariants, but kubernetes is still remarkably consistent in its api despite shortcomings of the language. And we'll show some good examples as we go along.-->

<!--OTE: i'll try to use "WE" and "OUR" for the needs of kube-rs)-->

---
### Kubernetes Invariants

- [apimachinery/meta/v1/types.go](https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.g)
- [client-go/kubernetes/typed](https://github.com/kubernetes/client-go/tree/master/kubernetes/typed)
- [kubernetes.io/docs/concepts](https://kubernetes.io/docs/concepts/)

notes:
- Let's talk about what kubernetes actually provides.
- these in particular
- start by diving into the arguably most important file of all
---
#### types.go: TypeMeta

[types.go#L36-56](https://github.com/kubernetes/apimachinery/blob/945d4ebf362b3bbbc070e89371e69f9394737676/pkg/apis/meta/v1/types.go#L36-L56)

```go
type TypeMeta struct {
    // +optional
    Kind string `json:"kind,omitempty" protobuf:"bytes,1,opt,name=kind"`
    // +optional
    APIVersion string `json:"apiVersion,omitempty" protobuf:"bytes,2,opt,name=apiVersion"`
}
```

notes:
- Every object has kind + version - flattened into the root structure

---
#### types.go: ObjectMeta
[types.go#L108-L282](https://github.com/kubernetes/apimachinery/blob/945d4ebf362b3bbbc070e89371e69f9394737676/pkg/apis/meta/v1/types.go#L108-L282)

<!--
    GenerateName string
    // read only
    UID types.UID
    ResourceVersion string
    Generation int64
    CreationTimestamp Time
    DeletionTimestamp *Time
    DeletionGracePeriodSeconds *int64
-->
```go
type ObjectMeta struct {
    Name string
    Namespace string

    Labels map[string]string
    Annotations map[string]string
    OwnerReferences []OwnerReference
    Finalizers []string
    ClusterName string
    ManagedFields []ManagedFieldsEntry
}
```

notes:
- Core metadata everyone thinks about. Simplified view, hidden read-only properties, annotations, everything is optional. Every object MUST have it, and must look like this.
- OwnerReferences, labels, annotations, finalizers, all great, managed fields (shrug) all that can go in there, and they're standardised.

---
#### types.go: List
[types.go#L913-L923](https://github.com/kubernetes/apimachinery/blob/945d4ebf362b3bbbc070e89371e69f9394737676/pkg/apis/meta/v1/types.go#L913-L923)

```go
type List struct {
    TypeMeta `json:",inline"`
    ListMeta `json:"metadata,omitempty"`
    Items []runtime.RawExtension `json:"items"`
}
```

notes:
- For when you ask for a collection of items (this contains `ListMeta` a much smaller variant that can contain continuation point and a remaining item count).
- More importantly; look at items there; a dynamic collection so this struct can be re-used.

---
#### types.go: APIResource
[types.go#L998-L1032](https://github.com/kubernetes/apimachinery/blob/945d4ebf362b3bbbc070e89371e69f9394737676/pkg/apis/meta/v1/types.go#L998-L1032)

```go
type APIResource struct {
    Name string
    SingularName string
    Namespaced bool
    Group string
    Version string
    Kind string
    Verbs Verbs
    ShortNames []string
    Categories []string
    StorageVersionHash string
}
```

notes:
- standardising where we we can get information of what Kind

---
#### types.go: ListOptions
[types.go#L328-L412](https://github.com/kubernetes/apimachinery/blob/945d4ebf362b3bbbc070e89371e69f9394737676/pkg/apis/meta/v1/types.go#L328-L412)

```go
type ListOptions struct {
    TypeMeta
    LabelSelector string
    FieldSelector string
    Watch bool
    AllowWatchBookmarks bool
    ResourceVersion string
    ResourceVersionMatch ResourceVersionMatch
    TimeoutSeconds *int64
    Limit int64
    Continue string
}
```

notes:
- All API params: GetOptions, ListOptions, DeleteOptions, PatchOptions.
- All parameters that the API accepts encapsulated into common structs from this root file.
- Error responses.
- LabelSelectors sitting inside ListOptions, so there's a generic way of filtering

---
#### Types.go

- 339 lines of code
- 928 lines of comments

notes:
- all this in 300 lines of code
- So I am raving this about this, but it's because of the consistency and complete adoption of everything in this file; that kubernetes feels so consistent and why we can actually make generic assumptions in other languages.
- now, writing structs is one thing, but how do we ensure these are consistent reused? lets look at client-go for a size contrast.

---
#### client-go: Deployment
[deployment.go#L41-L55](https://github.com/kubernetes/client-go/blob/36233866f1c7c0ad3bdac1fc466cb5de3746cfa2/kubernetes/typed/apps/v1/deployment.go#L41-L55)

```go
type DeploymentInterface interface {
    Create(ctx context.Context, deployment *v1.Deployment, opts metav1.CreateOptions) (*v1.Deployment, error)
    Update(ctx context.Context, deployment *v1.Deployment, opts metav1.UpdateOptions) (*v1.Deployment, error)
    UpdateStatus(ctx context.Context, deployment *v1.Deployment, opts metav1.UpdateOptions) (*v1.Deployment, error)
    Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error
    DeleteCollection(ctx context.Context, opts metav1.DeleteOptions, listOpts metav1.ListOptions) error
    Get(ctx context.Context, name string, opts metav1.GetOptions) (*v1.Deployment, error)
    List(ctx context.Context, opts metav1.ListOptions) (*v1.DeploymentList, error)
    Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error)
    Patch(ctx context.Context, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions, subresources ...string) (result *v1.Deployment, err error)
    GetScale(ctx context.Context, deploymentName string, options metav1.GetOptions) (*autoscalingv1.Scale, error)
    UpdateScale(ctx context.Context, deploymentName string, scale *autoscalingv1.Scale, opts metav1.UpdateOptions) (*autoscalingv1.Scale, error)
```

notes:
- typed api methods in client go (just the interface)
- getters/updaters/patchers/replacers/listers/deleters/watchers
- 200 line file for this object
- go to pod, show same except subresouce and object it acts on

---
#### client-go: Pod
[pod.go#L39-L54](https://github.com/kubernetes/client-go/blob/36233866f1c7c0ad3bdac1fc466cb5de3746cfa2/kubernetes/typed/core/v1/pod.go#L39-L54)

```go
type PodInterface interface {
    Create(ctx context.Context, pod *v1.Pod, opts metav1.CreateOptions) (*v1.Pod, error)
    Update(ctx context.Context, pod *v1.Pod, opts metav1.UpdateOptions) (*v1.Pod, error)
    UpdateStatus(ctx context.Context, pod *v1.Pod, opts metav1.UpdateOptions) (*v1.Pod, error)
    Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error
    DeleteCollection(ctx context.Context, opts metav1.DeleteOptions, listOpts metav1.ListOptions) error
    Get(ctx context.Context, name string, opts metav1.GetOptions) (*v1.Pod, error)
    List(ctx context.Context, opts metav1.ListOptions) (*v1.PodList, error)
    Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error)
    Patch(ctx context.Context, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions, subresources ...string) (result *v1.Pod, err error)
    GetEphemeralContainers(ctx context.Context, podName string, options metav1.GetOptions) (*v1.EphemeralContainers, error)
    UpdateEphemeralContainers(ctx context.Context, podName string, ephemeralContainers *v1.EphemeralContainers, opts metav1.UpdateOptions) (*v1.EphemeralContainers, error)
```

notes:
- same story for every object
- so.. there's a 200 line file for object
- Q: how could be this possibly be consistent? A: in the header

---
#### client-go: header
[deployment.go#L41-L55](https://github.com/kubernetes/client-go/blob/36233866f1c7c0ad3bdac1fc466cb5de3746cfa2/kubernetes/typed/apps/v1/deployment.go#L41-L55)

```go
// Code generated by client-gen. DO NOT EDIT.

package v1
```

notes:
- all of this is generated.
- and, it might seem obvious, you **have** to enforce some of these assumptions for them to stick, but it's still kind of crazy
- it's literally manual generics, with a bunch of glue to make it work.
- but it's consistent. for each, kind, the specific structs are specialized via external code generation, and the gen. source is present in repo

---
#### client-go

- tons of generated code per object
- [specialized client api](https://github.com/kubernetes/client-go/tree/master/kubernetes/typed)
- [specialized informers](https://github.com/kubernetes/client-go/blob/master/informers/apps/v1/statefulset.go#L58-L78)
- more than 100K lines of code  <!-- .element: class="fragment" -->

notes:
- much code
- client api, also informers for every object, client setup per group
- NEXT: as a result; client-go > 100K LOC (without vendoring)
- and i'm not at all passing judgement at this. this is great.
the fact that everything looks the same in here, is what enables `kubectl` to provide such a consistent interface, even if the language makes it hard for you to do so.
- moving on to documented concepts

---
### kubernetes.io: api endpoints

[api-concepts#standard-api-terminology](https://kubernetes.io/docs/reference/using-api/api-concepts/#standard-api-terminology)

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


notes:
- url consistency lets us make easy mappings between types and urls
- though things start to break down a little bit
- because this does not hold for pods, nodes, namespaces, service, pvcs, secret, or any other type in the core/v1 list. They have a different url that starts with `api` rather than `apis` + group missing

---
#### Broken: empty api group

```
GET /api/v1/pods

       !=

GET /apis/core/v1/pods
```


notes:
- it's a relatively minor inconsistency, coz we can just special case the empty group or core, but it's still awkward.

---
## kubernetes.io: watch events

[api-concepts#efficient-detection-of-changes](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)

```json
{ "type": "ADDED", "object": { \
    "kind": "Pod",  "apiVersion": "v1", \
    "metadata": {"resourceVersion": "10596", ...}, ...} }
{ "type": "MODIFIED", "object": { \
    "kind": "Pod", "apiVersion": "v1", \
    "metadata": {"resourceVersion": "11020", ...}, ...} }
```

notes:
- WatchEvs are what you receiv when you perform a watch call on any list EP
- this is how it looks (this response contains two lines)
- you'll get a chunked response, typically 1 line per chunk, but you'll have to buffer yourself until you have a complete line, because each of these lines can exceed the MTU
- but then for each line, you can parse the inner object as the type you actually want
- all apis use this and it's consistent -> source


---
## kubernetes.io: watch events - source

- [apimachinery:watch/watch.go#L40-L70](https://github.com/kubernetes/apimachinery/blob/681a08151eac875afc5286670195105118d3485d/pkg/watch/watch.go#L40-L70)
- [apimachinery:meta/watch.go#L31-L40](https://github.com/kubernetes/apimachinery/blob/594fc14b6f143d963ea2c8132e09e73fe244b6c9/pkg/apis/meta/v1/watch.go#L31-L40)

```go
const (
    Added    EventType = "ADDED"
    Modified EventType = "MODIFIED"
    Deleted  EventType = "DELETED"
    Bookmark EventType = "BOOKMARK"
    Error    EventType = "ERROR"
)

type WatchEvent struct {
    Type string `json:"type"`
    Object runtime.RawExtension `json:"object"`
}
```
notes:
- we find more runtime generics.
- covered all concepts and main api consistencies now
- so the rest will be more from a rust POV

---
### Rust Modelling

<ul>
    <li class="fragment"><a href="https://github.com/clux/kube-rs/">clux/kube-rs</a></li>
    <li class="fragment">Arnav Singh / @Arnavion - <a href="https://github.com/Arnavion/k8s-openapi">k8s-openapi</a></li>
</ul>

<small class="fragment">Bryan Liles: <a href="https://youtu.be/Rbe0eNXqCoA?t=566">client-go is not for mortals</a></small>

notes:
- like the go code, will be slightly simplifying for readability, and most of the stuff here is kube-rs
- but start out with code in a project by Arnav Singh aka Arnavion
- the project really is the lynchpin that makes any generics possible
- generates rust structures from openapi schemas, plus factoring out some of "the consistency" into a few traits that is then implemented for these structures
- so huge shoutout to him. for what i believe is just his side project, i really cannot thank him enough
- motivation a bit out of need - but also partly out of the call to action by bryan liles at kubecon barcelona

---
### k8s-openapi: Resource Trait

```rust
pub trait Resource {
    const API_VERSION: &'static str;
    const GROUP: &'static str;
    const KIND: &'static str;
    const VERSION: &'static str;
}
```

notes:
- TL;DR: A rust trait is "behaviour" you can implement for a type, and then later you can use that trait as a constraint in function signatures
- Generally "behaviour", can't put dynamic data in them, but you are allowed to put in static associated constants.
- so we can use this to map an object to where **on** the api it lives.

---
### k8s-openapi: Metadata Trait

```rust
pub trait Metadata: Resource {
    fn metadata(&self) -> &ObjectMeta;
}
```

notes:
- Trait is just a way to grab metadata that is consistent across all objects.
- Even if always on same key, type system can't guarantee that.
- Slightly simplifying; k8s-openapi distinguishes between listable types using `ListMeta`, but everything else returns `ObjectMeta`
- and we (kube-rs) can only really do useful ops on top of objects that have `ObjectMeta`, so slightly hiding a few details.

---
### kube-rs: Resource struct

```rust
pub struct Resource {
    pub api_version: String,
    pub group: String,
    pub kind: String,
    pub version: String,
    pub namespace: Option<String>
}
```

notes:
- Got two root traits. Let's build a dynamic api on top of them.
- You may note that this is basically a dynamic version of the `Resource` trait, but it allows carrying the dynamic namespace property and can be instantiated at runtime from an arbitrary object (helpful for CRDs).
- We *CAN* fill these in at runtime, but for existing openapi structs, can get a blanket ctor with one trait constrait:

---
### kube-rs: Resource namespaced ctor

```rust
use k8s_openapi::Resource as ResourceTrait;

impl Resource {
    pub fn namespaced<K: ResourceTrait>(ns: &str) -> Self {
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

Notes:
- All the data, except namespace, is already on the trait, so we just constrain by that
- NB: Resource type is not generic, but this particular ctor is.
- With this we can hit every objects api endpoints. Demonstrate: mapper

---
### kube-rs: Url mapper

```rust
impl Resource {
    fn make_url(&self) -> String {
      format!("/{group}/{api_version}/{namespaces}{resource}",
        group = if self.group.is_empty() {"api"} else {"apis"},
        api_version = self.api_version,
        resource = to_plural(&self.kind.to_ascii_lowercase()),
        namespaces = self.namespace.as_ref()
          .map(|n| format!("namespaces/{}/", n))
          .unwrap_or_default())
    }
}
```

notes:
- function that dictates all of k8s urls on top of this struct
- handles that special empty group case
- CAVEAT: due to limitation of the trtait: load-bearing pluralize.
phrase i had never believed i had to use to describe software architecture, let alone from my own designs, but here we are.
- ..but with url mapper implement => we CAN MAKE DYNAMIC API

---
### kube-rs: Dynamic API

```rust
impl Resource {
    pub fn create(&self, pp: &PostParams, data: Vec<u8>)
        -> Result<Request<Vec<u8>>>
    {
        let base_url = self.make_url() + "?";
        let mut qp = Serializer::new(base_url);
        if pp.dry_run {
            qp.append_pair("dryRun", "All");
        }
        let urlstr = qp.finish();
        let req = http::Request::post(urlstr);
        req.body(data).map_err(Error::HttpError)
    }
}
```

notes:
- Create as ex. for basic crud.
- Takes one of the PostParam structs (types.go), binary data, makes qp from postparams, and preps request. You must execute yourself. Sans-io.
- This is now something similar to other language clients. Bytes come in, goes through a url mapper and an http call, and response bytes come out.
- Of course, this isn't really what we want. We don't want to be interjecting at every point, to try deserialize a bytestream into a concrete type.
- What we really want, is automatic ser/de-ization, and a mechanism generic over K that is aware of underlying struct for the resource.

---
### kube-rs: Typed API

```rust
pub struct Api<K> {
    resource: Resource,
    client: Client,
    phantom: PhantomData<K>,
}

let api: Api<Pod> = Api::namespaced(client, ns);
```

notes:
- For that we our first truly generic type. It's a wrapper around a resource, with an http client handle inside of it, along with an empty marker of what type it's for.
- Don't actually store data related to K, so just a marker for typesystem.
- need to coerce to K somewhere, so should probably be at ctor.
- Can make Api::namespaced by referencing Resource::namespaced
- can create an Api (Client), and tell it at ctor time, that it's for Pods.
- Now Let's generalize create.

---
### kube-rs: Typed API methods

```rust
impl<K> Api<K>
where K: Clone + Deserialize + Metadata,
{
    pub async fn create(&self, pp: &PostParams, data: &K)
        -> Result<K>
    where K: Serialize,
    {
        let bytes = serde_json::to_vec(&data)?;
        let req = self.resource.create(&pp, bytes)?;
        self.client.request::<K>(req).await
    }
}
```

notes:
- weird syntax? generic impls, K needs to satisfy constraints
- K needs extra constraints for one method
- Uses serialize trait, and tells client to execute req and deserialize
- By using generics and constraints on `K` we have implement this `client-go` like api method, across all types just a single blanket impl.
- Great, but can generics solve everything? Won't we still need codegen?

---
### Code Generation

<ul>
    <li class="fragment">first class integration via cargo build</li>
    <li class="fragment"><a href="https://doc.rust-lang.org/reference/procedural-macros.html">procedural macros</a></li>
    <li class="fragment">#[derive(CustomTrait)]</li>
    <li class="fragment">#[custom_trait_attr]</li>
<!--- cargo expand-->
</ul>

notes:
- Yes, code generation still happens in rust. But it's a required part of cargo build to execute.
- Called proc macros, and I like to desc as "compile time decorators"
- user interface to them is super compelling, though tricky to write
- But because of that first class support for code generation, a whole class of errors where you are operating on a stale version of generated code, is now elimiated. The compiler disallows that possibility.

---
### Serialize
<!--USER FACING CODE STARTS HERE-->

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct FooSpec {
    name: String,
    is_bad: Option<String>,
}
```

notes:
- First example everyone sees is derive `Serialize` + `Deserialize` from the `serde` library. Implements these traits and allows you to convert between various serialization formats. Can customize it further with struct level, or field level attributes.
- In practice, you often end up writing much of the same gunk/annotations as you would with go's json encoding to like distinguish casings of your code and disk format, but there's type safety + error handling around it.

---
### kube-derive: CustomResource

```rust
#[derive(CustomResource, Serialize, Deserialize, Clone)]
#[kube(group = "clux.dev", version = "v1", kind = "Foo")]
#[kube(namespaced, status = "FooStatus")]
pub struct FooSpec {
    name: String,
    info: Option<String>,
}
```

notes:
- We can also make our own derivable with our own options for it. Here we are using kube's `CustomResource` proc-macro, and we are telling kube, the values of the resource parameters (group, version, kind). This will create all the code around a custom resource.
- Bunch more options available, we've tried to mimic the usability of kubebuilder setup in this particular case.
- What it does: creating a type Foo attaching spec/status/meta, ctor, crd method.

---
### Example: Using a CRD

```rust
let crds: Api<CustomResourceDefinition> = Api::all(client);
crds.create(&pp, &Foo::crd()).await;

let foos: Api<Foo> = Api::namespaced(client, &namespace);

let f = Foo::new("eirik-example", FooSpec {
    name: "i am a foo crd instance".into(),
    info: None
});
let o = foos.create(&pp, &f2).await?;
```

notes:
- The generated `Foo` type (containing metadata, spec, pointing to your spec, etc), also has a `crd` method. So you can literally just apply it and start using it in like `main`.
- ideally, error handle and use server side apply. illustrative.
- also, only really covered Api::create

---
### WatchEvent

```rust
#[derive(Deserialize, Serialize, Clone)]
#[serde(tag = "type", content = "object")]
#[serde(rename_all = "UPPERCASE")]
pub enum WatchEvent<K> {
    Added(K),
    Modified(K),
    Deleted(K),
    Bookmark(Bookmark),
    Error(ErrorResponse),
}
```

notes:
- NOW. go beyond basic crud and into watch land.
- 1st WE: maps nicely one in apimachinery that contained the dynamic runtime object.
- In rust, it can be packed into a generic enum, for a fully typed one. Great.
- The serde tags here tells serde that the values in enum variants -> in object key, and enum variant name -> in tag key (tags sent/recvd as uppercase to match go convention).
- ..This is what watch would return...

---
### Watch

```rust
impl<K> Api<K>
where K: Clone + Deserialize + Metadata,

    pub async fn watch(&self, lp: &ListParams, rv: &str)
        -> Result<impl Stream<Item = Result<WatchEvent<K>>>>
    {
        let req = self.resource.watch(&lp, &rv)?;
        self.client.request_events::<K>(req).await
    }
}
```

notes:
- ..well. Significantly more intimidating signature.
- have type that contain impl Stream => constraint says the return type must implement the Stream trait.
- Stream == async iterator. Have to await each new element.
- Element? WatchEvent that can fail <- Stream of
- stream is also wrapped in result because HTTP req can fail, so that has to succeed before you can start streaming
- fairly chonky type
- looks hard immediately, and haven't even talked about the corner cases..

---
### Broken: Watch

<ul>
<li class="fragment">resourceVersion bookkeeping</li>
<li class="fragment">stale resourceVersions <a href="https://github.com/kubernetes/kubernetes/issues/87292">#87292</a></li>
<li class="fragment">5 minute max limit <a href="https://github.com/kubernetes/kubernetes/issues/6513">#6513</a></li>
<li class="fragment">large data use <a href="https://github.com/kubernetes/kubernetes/issues/90339">#90339</a>, <a href="https://github.com/kubernetes/kubernetes/issues/82655">#82655</a></li>
</ul>

notes:
- Gotta Track ResourceVersions; integers passed on via etcd, must pass these on for every watch call, to tell k8s where you left off.
- Sometimes these RVs are stale, and if you are building a state cache like a reflector, you have to re-list and get all the state back for every object in the system if you get desynchronized. Before bookmarks, that was very likely to happen.
- Watch calls also can't reliably stay open for more than 5 minutes, so you have to keep issuing this watch call at least that frequently.
- and finally, sheer data use of it. On EVERY CHANGE incl status. Seen NodeStatus, last updated timestamps inside conditions? Every few seconds, you'll get the whole heckin' object. (Can hide, but still networked)
- => anyone building a controller type solution will need abstractions.

---
### watcher abstraction

<ul class="fragment">
  <li>LIST</li>
  <li>stream</li>
  <li>handle stream errors behind the scenes</li>
  <li>maybe RE-LIST (duplicate + dropped events)</li>
  <li>propagate user errors</li>
  <li>only propagate events</li>
</p>

notes:
- what would such an abstraction do?
- Well we got to watch continously, but not longer than 5 minutes, propagate all user errors, re-list on desync errors, and still somehow encapsulate it all in one nice stream. It's absolutely not trivial.

---
### kube-runtime

<ul>
  <li class="fragment">Teo K. Röijezon - <a href="https://github.com/teozkr/">teozkr</a></li>
  <li class="fragment">Entirely Stream based solution</li>
  <li class="fragment">watcher</li>
  <li class="fragment">reflector with Store</li>
  <li class="fragment">Controller</li>
</ul>

notes:
- So a huge shoutout to my other maintainer: Teo.
- He basically figured out an entirely Stream based solution for (not only) watchers, but also reflectors and controllers
- and because these objects are just this rust native concept of a stream, they end up being possible to manipulate in very standard ways; store, pass around, extend, integrate, instrument, test
- we've not gotten around to showcase, nor poc all of that, and this definitely has rough edges, but it's definitely the best evolution point so far for a controller-runtime in rust
- so will quickly go through how they work

---
### kube-runtime: watcher

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

```rust
watcher(api, listparams)
    -> impl Stream<Item = Result<watcher::Event<K>>>
```

notes:
- Funnily enough, watchers end up being one of the more complicated of the three. Entirely due to watch corner cases.
- Internally, we model it with FSM. And we are using basically a state transformer to pass around the STATE (shown above), along with the actual watch events (ultimate thing we want to return)
- But because one of these could be a list step, a watcher::Event could be a whole collection - so we expose helpers that flatten this into a regular watchevent stream

---
### kube-runtime: watcher usage

```rust
let cms: Api<ConfigMap> = Api::namespaced(client, &namespace);
let lp = ListParams::default();

let mut w = try_flatten_applied(watcher(cms, lp)).boxed();
while let Some(event) = w.try_next().await? {
    info!("Got: {:?}", event);
}
```

notes:
- suppose i only want to subscribe to Added or Modified ew for ConfigMaps, in some namespace, this is how that would look. that could basically be your main.
- line 4; watcher on configmaps, flatten and filter to applied events
- handles all the watch complexity

---
### kube-runtime: reflector


```rust
pub fn reflector<K, W>(mut store: Writer<K>, stream: W)
    -> impl Stream<Item = W::Item>
where
    K: Metadata + Clone,
    W: Stream<Item = Result<watcher::Event<K>>>,
{
    stream.inspect_ok(move |event| {
        store.apply_watcher_event(event)
    })
}
```


notes:
- reflector is a watcher that stores the result of events in a store.
- and that description can be translated into a single line body
- observe a watcher stream, and you inserting/replacing/removing objects from store, then pass on the events unmodified
- complicated signature, you need a Store<K> (which i've not defined, but hashmap aware of watchevents)
- you need the unflattend stream that the watcher is outputting; that's W


---
### kube-runtime: reflector usage

```rust
let cms: Api<ConfigMap> = Api::namespaced(client, &namespace);

let writer = Writer::<ConfigMap>::default();
let reader = writer.as_reader();
let rf = reflector(writer, watcher(cms, lp));

let mut w = try_flatten_applied(rf).boxed();
while let Some(event) = w.try_next().await? {
    info!("Applied {}", Meta::name(&event));
}
```

notes:
- To use this you construct a writer, and a watcher. Then you use is as a watcher, like at the end
- More importantly; 3 lines center; You can get a reader from the writer, and use that as state in a like a web framework. Can be cloned.
- What is not clonable; the writer. Because it's unsound to have multiple things writing to the same store. So that has to be illegal.
- By illegal; Don't mean we force this condition on an unknowning user at the last line of your (go) doc.
- I mean; it's actually a compile error to try to use the writer after making the reflector. Thanks to move semantics.
- Move on to the big one. Controller

---
### kube-runtime: Controller

```rust
#[tokio::main]
async fn main() -> Result<(), kube::Error> {
    let client = Client::try_default().await?;
    let context = Context::new(());
    let cmgs = Api::<ConfigMapGenerator>::all(client.clone());
    let cms = Api::<ConfigMap>::all(client.clone());

    Controller::new(cmgs, ListParams::default())
        .owns(cms, ListParams::default())
        .run(reconcile, error_policy, context)
        .await;
    Ok(())
}
```

notes:
- C is a system reconciles an object/CR along with child objects it owns - calls a reconcile fn when anything related changes.
- That's it's job, combining input streams, debouncing events, scheduling retries.
- builder pattern: should remind you a bit of controller-runtime. heavily inspired (got help).
- ex: CMG ensuring CM is in correct state
- completely sufficient main
- not shown: you derive your CR from a struct (shown), and provide error handling policy, plus a reconciler fn, that will be called with a context you can define

---
### kube-runtime: Controller reconciler

```rust
async fn reconcile(cmg: ConfigMapGenerator, ctx: Context<()>)
        -> Result<ReconcilerAction, Error>
{
    // TODO: update CM to match cmg.content
    // TODO: update CMG.status
    Ok(ReconcilerAction {
        requeue_after: Some(Duration::from_secs(300)),
    })
}
```

notes:
- How reconcile looks. If you need access to anything here you can stuff it into your context.
- In the interest of not obscuring the slide; this fn is where you would grab a client from the Context, and start making api calls to k8s to ensure CM is up to date with GEN. Write to status object to indicate how far you got.

---
### Building Controllers

- controller-runtime advice applies <!-- .element: class="fragment" -->
- idempotent, error resilient reconcilers <!-- .element: class="fragment" -->
- use server side apply <!-- .element: class="fragment" -->
- use finalizers <!-- .element: class="fragment" -->

notes:
- seems handwavey, but not going to rehash best practices for writing controllers here
- most advice from kubebuilder / controller-runtime generally applies (talks)
- TL;DR: reconcile needs to be idempotent, check state of the world before you redo all the work on a duplicate event. use server side apply.
- use finalizers to gc. If you control an object, put an ownerreference on it.

---
### Examples

<ul>
    <li class="fragment"><a href="https://github.com/clux/controller-rs">controller-rs</a> and <a href="https://github.com/clux/version-rs">version-rs</a></li>
    <li class="fragment">Bring your own deps</li>
    <li class="fragment">web: <a href="https://crates.io/crates/actix-web">actix-web</a>, <a href="https://crates.io/crates/warp">warp</a>, <a href="https://crates.io/crates/rocket">rocket</a></li>
    <li class="fragment">o11y: <a href="https://crates.io/crates/tracing">tracing</a>, <a href="https://crates.io/crates/sentry">sentry</a>, <a href="https://crates.io/crates/prometheus">prometheus</a></li>
</ul>

notes:
- basic setup for how things work
- No scaffolding here. Choose your own dependencies.
- Frameworks? Maybe you want one, good practice to expose metrics.
- o11y: tracing eco really solid - slap on a #[instrument] proc macro, and add your favourite tracing subscriber
- sentry for error reporting, or prometheus for custom metrics

---
### End

- Eirik Albrigtsen
- [clux](https://github.com/clux) / [@sszynrae](https://twitter.com/sszynrae)
- [kube-rs](https://github.com/clux/kube-rs)

notes:
- that's it.
- We're doing this because we want something: light weight, easy to understand. Not much indirection. No crazy scaffolding. And type safety.
- Api crate (kube) quite stable, but kube-runtime is pretty new still, so anyone that's willing to get their hands dirty, help is appreciated.
