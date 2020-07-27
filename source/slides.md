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
- We'll identify some of these invariants.
- Then talk about how to model the same api in rust using generics, and see that it gives us the same consistency more-or-less for free.
- We'll also touch on async api design in rust during this modelling process. Async rust was only properly released about a year ago, and the rust ecosystem has consequently seen enormous advances in this year with it stable. So if you're not up to speed, you'll at least see some patterns in this talk.


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
- OwnerReferences, labels, annotations, finalizers, managed fields that all can go in there, and they're standardised.

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
- lets look at client-go for a contrast

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
- typed api methods in client go
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
- because people recognised that you **have** to enforce some of these assumptions for them to stick.
- now, this isn't generics, but it's consistency. for each, kind, the specific structs are specialized via external code generation - but the gen. source is present in repo regardless

---
#### client-go

- tons of generated code per object
- [specialized client api](https://github.com/kubernetes/client-go/tree/master/kubernetes/typed)
- [specialized informers](https://github.com/kubernetes/client-go/blob/master/informers/apps/v1/statefulset.go#L58-L78)
- more than 100K lines of code  <!-- .element: class="fragment" -->

notes:
- much code
- client api, also informers for every object
- as a result; client-go > 100K LOC (without vendoring)
- and i'm not trying to judge here. this is great.
the fact that everything looks the same in here, is what enables `kubectl` to provide such a consistent interface.

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
- it also returns the etcd resource version we last saw (resume point in some sense)
- all apis use this and it's consistent.


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
- from source, more runtime generics.

---
### Rust Modelling

- [kube-rs](https://github.com/clux/kube-rs/)
- Arnav Singh / @Arnavion - [k8s-openapi](https://github.com/Arnavion/k8s-openapi)

notes:
- at this point we have actually covered all the core ideas we need to talk about this from the rust POV
- and the rest of the talk will feature a grab bag of different rust code, shown here in slightly simplified code here, much of which are from kube-rs
- but also huge shoutout to Arnav Singh
- the project really is the lynchpin that makes any generics possible
- generates rust structures from openapi schemas, plus factoring out some of "the consistency" into a few traits that is then implemented for these structures

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
- TL;DR: A rust trait is behaviour you can implement for a type, and then later you can use that trait as a constraint in function signatures
- Normally traits are meant to encapsulate behaviour, can't put dynamic data in them, but you are allowed to put in static associated constants.
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
- For this to be implementable the type kind of needs to be the same.
- Slightly simplifying; as the actual one is slightly more general, and allows parametrising the metadata types. Not super relevant, but: all listable types uses `ListMeta`, but everything else returns `ObjectMeta`
- But we (kube-rs) can only really do useful ops on top of objects that have `ObjectMeta`, so theres' slightly more indirection for us to actually get the the behaviour we want.

---
### kube-rs: Resource struct

```rust
#[derive(Clone, Debug)]
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
- Don't copy paste. Canadian Aboriginal Syllabics coz reveal tried to close my html string tag.
- You may note that this is basically a dynamic version of the `Resource` trait, but it allows carrying the dynamic namespace property and can be instantiated at runtime from an arbitrary object (helpful for CRDs).
- For CRDs we can create this manually, with like a builder, but for existing openapi structs can get a blanket ctor with one trait constrait:

---
### kube-rs: Resource namespaced ctor

```rust
impl Resource {
    pub fn namespacedᐸK: k8s_openapi::Resourceᐳ(ns: &str) -> Self {
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
- This constraint does not require the struct to implement the trait, it just needs it for that quick constructor

---
### kube-rs: Url mapper

```rust
impl Resource {
  fn make_url(&self) -> String {
    let ns = self.namespace.as_ref()
        .map(|n| format!("namespaces/{}/", n));
    let plural = to_plural(&self.kind.to_ascii_lowercase());
    format!(
        "/{group}/{api_version}/{namespaces}{resource}",
        group = if self.group.is_empty() {"api"}else{"apis"},
        api_version = self.api_version,
        namespaces = ns.unwrap_or_default(),
        resource = plural,
    )
  }
}
```

notes:
- We can also create the function that dictates all of k8s urls on top of this struct
- handles that special empty group case
- CAVEAT: due to limitation of the trtait: load-bearing pluralize.
phrase i had never believed i had to use to describe software architecture, let alone from my own designs, but here we are.
- ..but with url mapper => we CAN MAKE DYNAMIC API

---
### kube-rs: Dynamic API

```rust
impl Resource {
    pub fn create(&self, pp: &PostParams, data: Vecᐸu8ᐳ)
        -> ResultᐸRequestᐸVecᐸu8ᐳᐳᐳ
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
- This is now something similar to other language clients. Bytes come in, goes through a url mapper and an http call, and response bytes come out.
- Of course, this isn't really what we want. We don't want to be interjecting at every point of the way to try to deserialize a bytestream into a concrete type.
- What we really want, is automatic serialization of an instantiated object, and automatic deserialization of the response type into the correct object.

---
### kube-rs: Typed API

```rust
pub struct ApiᐸKᐳ {
    resource: Resource,
    client: Client,
    phantom: PhantomDataᐸKᐳ,
}

let api: ApiᐸPodᐳ = Api::namespaced(client, ns);
```

notes:
- For that we our first truly generic type. It's a wrapper around a resource, and we put a copy of a http client inside of it, along with an empty marker of what type it's for (need to coerce somewhere - api only handles one object type)
- But notice there were no constraints on `K` here - they come in impls

---
### kube-rs: Typed API methods

```rust
implᐸKᐳ ApiᐸKᐳ
where K: Clone + Deserialize + Metadata,
{
    pub async fn create(&self, pp: &PostParams, data: &K)
        -> ResultᐸKᐳ
    where K: Serialize,
    {
        let bytes = serde_json::to_vec(&data)?;
        let req = self.resource.create(&pp, bytes)?;
        self.client.request::ᐸKᐳ(req).await
    }
}
```

notes:
- weird syntax, generic impls, K needs to satisfy constraints
- K needs extra constraints for one method
- By adding constraints on `K` we can implement `client-go` like methods on this ad-hoc `Api` struct across all types openapi generated types with a single blanket impl.

<!--
#### SKIIIP Broken: spec/status

```rust
impl<K> Api<K>
where
    K: Clone + DeserializeOwned,
{
    pub async fn patch_status(&self, name: &str, pp: &PatchParams, patch: Vec<u8>) -> Result<K> {
        let req = self.resource.patch_status(name, &pp, patch)?;
        self.client.request::<K>(req).await
    }
}
```

Remember how we couldn't rely on spec/status? Well, this means that we can't take a generic `Status` type atm. You have to supply something you serialize yourself, and hope wait for kubernetes to validate your object.

Wait, why? Well, let's try to replicate the object model first.

#### Broken: Object<Spec, Status>
Presumably many of you have seen this representation.

```rust
pub struct Object<Spec, Status> {
    pub types: TypeMeta, // apiVersion + kind
    pub metadata: ObjectMeta,
    pub spec: Spec,
    pub status: Option<Status>,
}
```

CLAIM: A k8s object consists only `apiVersion` + `kind` (typemeta), `metadata`, `spec` and an optional `status` struct.

You very often hear this used as way to explain the core ideas of desired vs observed state. Even maintainers will use this simplification.

If we're omitting trait constraints, this is how it would look in rust. `Spec` and `Status` here are generic type parameters and are specialized at compile time for the various invocations.

..but the problem with this model, is ultimately that it not true in general.

### Broken: Snowflakes
Here are some openapi rust structs for what I like to call snowflakes.

```rust
pub struct ConfigMap {
    pub metadata: ObjectMeta,
    pub binary_data: Option<BTreeMap<String, ByteString>>,
    pub data: Option<BTreeMap<String, String>>,
    pub immutable: Option<bool>,
}
```

Look at configmap (data +  binary_data). Fields at the top level.

```rust
pub struct Secret {
    pub metadata: ObjectMeta,
    pub data: Option<BTreeMap<String, ByteString>>,
    pub immutable: Option<bool>,
    pub string_data: Option<BTreeMap<String, String>>,
    pub type_: Option<String>,
}
```

similar story for secret

```rust
pub struct ServiceAccount {
    pub metadata: ObjectMeta,
    pub automount_service_account_token: Option<bool>,
    pub image_pull_secrets: Option<Vec<LocalObjectReference>>,
    pub secrets: Option<Vec<ObjectReference>>,
}
```

..and serviceaccount. just a like 30 character long bool at the top level.
...tons more `Endpoint` (subsets vec), `Role` (rules obj), `RoleBinding` (subjects + roleRef).

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

or more amusingly, `Event`, with 15 at the root.

### What can we rely on?
So the core objects really cause a lot of trouble. Can't rely on SPEC/STATUS.
If we can't even rely on that..just how much of metadata can we rely on?

#### Broken: Optional metadata
screenshot code with the +optional... in pod?
https://github.com/kubernetes/api/blob/master/core/v1/types.go#L3667-L3686
...how? we said we had that guarantee?

we basically only see this as user input to `patch` requests that allow sending almost completely blank objects (and only spec or status, say). works because name of obj already inferrable from the url.

so this is one we deliberately disobey.
because it makes it so awkward to unwrap something that has to be there

#### Broken: Optional names
more like that? here's another property that feels mandatory: `metadata.name`
even though a resource having a name is a fundamental req for a get request:

metadata.name optional (`generatename` mechanism)
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L117-L118
makes sense, but now every clients now have to assume non-null

it's an easy assumption to make, but it's just one prominent example of many
and it leads you down a very uneasy road of not being able to assume anything.

so in general, we have to write awkward code and unpack optionals -_-

### TODO: Show api trait signatures?
-->



---
### In general: Lean on types
Lot of benefits to leaning on types. You write things once and it is used by everything. We want code to take effect immediately rather than have to step through a code generation pattern, and then commit generated code to a repo.

---
### Code Generation
But that's not to say we don't do code generation. Rust has procedural macros, which lets us do code generation at compile time with `cargo build` and this code is used in the later stages of the same compilation. So that first class support for code generation basically eliminates a whole class of errors where you are operating on a stale version of generated code, because the compiler disallows that possibility.


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
- Just the basic derives that almost everyone uses for `Serialize` and `Deserialize` from the `serde` library. This gives you serialization and deserialization methods that all follow standard traits.
- In practice, you often end up writing much of the same annotations as you would with go's json encoding to like distinguish casings of your code and disk format, but there's type safety around it. Not just comments in backticks.

## CustomResource

```rust
#[derive(CustomResource, Serialize, Deserialize, Clone)]
#[kube(group = "clux.dev", version = "v1", kind = "Foo", namespaced)]
#[kube(status = "FooStatus")
pub struct FooSpec {
    name: String,
    info: Option<String>,
}
```

notes:
- And we can also make our own derive rules and options for it. Here we are using kube's `CustomResource` proc-macro, and we are telling kube what the resource parameters are (group, version, kind). This will create all the code around a custom resource.
- We've tried to mimic some of the usability of kubebuilder here, but without any of the stored generated code.

---
### Example: Using a CRD

```rust
let crds: ApiᐸCustomResourceDefinitionᐳ = Api::all(client);
crds.create(&pp, &Foo::crd()).await;
let foos: ApiᐸFooᐳ = Api::namespaced(client, &namespace);

let f = Foo::new("eirik-example", FooSpec {
    name: "i am a foo crd instance".into(),
    info: None
});
let o = foos.create(&pp, &f2).await?;
```

notes:
- The generated `Foo` type (containing metadata, spec, pointing to your spec, etc), also has a `crd` method. So you can literally just apply it and start using it in like `main`.


<!--
### SKIP: Broken: Conditions
while we are talking about conditions
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L1367
sits inside a vector, so they all look the same, so you have to always filter the conditions for the type you want. presumably so that `kubectl describe` can display all conditions in one nice table.

but, we could deal with that. What we cannot deal with is that you cannot really `patch_status` to update particular condition entry.

none of the original patch types even work (strategic might have, but not supported on crds). so you need at least server side apply to even use conditions.

SKIP DUE TO https://github.com/clux/kube-rs/issues/43 FIXED IN SS APPLY?
-->

---
### Watch

```rust
    pub async fn watch(&self, lp: &ListParams, rv: &str)
        -> Resultᐸimpl StreamᐸItem = ResultᐸWatchEventᐸKᐳᐳᐳᐳ
    {
        let req = self.resource.watch(&lp, &rv)?;
        self.client.request_events::ᐸKᐳ(req).await
    }
```

notes:
- Talked about basic crud operations (same pricinple as `create`).
- One that is fundamentally different is watch. Watch is chunked. It's async.
- So watch returns a complicated dynamic type that implements the Stream trait (impl Stream). Stream == async iterator. Have to await each new element.
- Wrapped in result because HTTP req can fail, so if that succeeded then you are streaming

---
### Broken: Watch

- resourceVersion bookkeeping
- stale resourceVersions <!-- .element: class="fragment" -->
- 5 minute limit <!-- .element: class="fragment" -->
- large data use <!-- .element: class="fragment" -->

notes:
- Nice signature from that, BUT. Watch is awkward. ResourceVersions integers exposed via etcd, that you have to return on every watch call to tell k8s where you left off.
- Sometimes these RVs are stale, and if you are building a state cache like a reflector, you have to re-list and get all the state back for every object in the system if you get desynchronized. Before bookmarks, that was very likely to happen.
- Watch calls also can't reliably stay open for more than 5 minutes, so you have to keep issuing this watch call at least that frequently.

and finally, the obscene amount of data this can return. Tried using a node informer? insane amount of noise. FULL 10k data every 5s because the conditions in its status object contain a last updated timestamp...

---
### WatchEvent
That said, the `WatchEvent` itself is nice. Remember how watch events all packed an object inside of it? We can model this in rust with generic enums:

```rust
#[derive(Deserialize, Serialize, Clone)]
#[serde(tag = "type", content = "object", rename_all = "UPPERCASE")]
pub enum WatchEventᐸKᐳ {
    Added(K),
    Modified(K),
    Deleted(K),
    Bookmark(Bookmark),
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

---
## Runtime
How to build on top of watch and the api. Well we got to watch continously, but not longer than 5 minutes, propagate all user errors, retry/re-list on desync errors, and still somehow encapsulate it all in one nice stream. It's absolutely not trivial.

So a huge shoutout to my other maintainer:

- Teo K. Röijezon / @teozkr who wrote kube-runtime (controller-runtime equivalent)

He basically figured out an entirely Stream based solution for watchers/reflectors and controllers, and rewrote that entire module of `kube`.

It's an amazing technical achievement that makes it really easy to integrate into your application.

---
### Watcher
Informer-like. But FSM.

```rust
enum StateᐸK: Meta + Cloneᐳ {
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

---
### Reflector
Builds on top of watcher and adds a store.

```rust
let cms: ApiᐸConfigMapᐳ = Api::namespaced(client, &namespace);

let store = reflector::store::Writer::ᐸConfigMapᐳ::default();
let reader = store.as_reader();
let rf = reflector(store, watcher(cms, lp));
```

Move ensures no use after construction. Writer disappears. No weird contracts in godoc. Enforce it in the code.

what is a reflector?

```rust
pub fn reflectorᐸK: Meta + Clone, W: StreamᐸItem = Resultᐸwatcher::EventᐸKᐳᐳᐳᐳ(mut store: store::WriterᐸKᐳ, stream: W)
    -> impl StreamᐸItem = W::Itemᐳ
{
    stream.inspect_ok(move |event| store.apply_watcher_event(event))
}
```

---
### Controller
Controller is a system that calls your reconciler with events as configured.
You define 2 fns. One where you write idempotent (not going to talk about how to write resilient controllers, all normal advice (kbuilder etc) applies).
Second one is an error handler. You might want to check every error dilligently within the reconciler, but you can also just use `?`.

```rust
async fn reconcile(g: ConfigMapGenerator, ctx: Contextᐸ()ᐳ) -> Result<ReconcilerAction, Error> {
    // TODO: reconcile
    Ok(ReconcilerAction {
        requeue_after: Some(Duration::from_secs(300)),
    })
}
fn error_policy(_error: &Error, ctx: Contextᐸ()ᐳ) -> ReconcilerAction {
    // TODO: handle non-Oks from reconcile
    ReconcilerAction {
        requeue_after: Some(Duration::from_secs(60)),
    }
}
```

if you have those, then it's just hooking up events and contexts:

```rust
async fn main() -> Resultᐸ(), kube::Errorᐳ {
    let client = Client::try_default().await?;
    let context = Context::new(()); // bad empty context - put client in here
    let cmgs = Api::ᐸConfigMapGeneratorᐳ::all(client.clone());
    let cms = Api::ᐸConfigMapᐳ::all(client.clone());
    Controller::new(cmgs, ListParams::default())
        .owns(cms, ListParams::default())
        .run(reconcile, error_policy, context)
        .await;
    Ok(())
}
```

should remind you a bit of controller-runtime. heavily inspired (got help).

---
### Building Controllers
not rehashing best practices. most advice from kubebuilder / controller-runtime applies. reconcile needs to be idempotent, check state of the world before you redo all the work on a duplicate event. use server side apply. use finalizers to gc.

---
### Examples
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

---
### Caveats
Rough edges. Api library (kube) quite stable, but kube-runtime is pretty new still. Show users and testimonials. Kruslet.

Vision: light weight, easy to understand. Not much indirection. No crazy scaffolding. And type safety.
Rust ideal for this, but we are in early stages.
