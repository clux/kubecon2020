
####Broken: spec/status

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


### SKIP: Broken: Conditions
while we are talking about conditions
https://github.com/kubernetes/apimachinery/blob/master/pkg/apis/meta/v1/types.go#L1367
sits inside a vector, so they all look the same, so you have to always filter the conditions for the type you want. presumably so that `kubectl describe` can display all conditions in one nice table.

but, we could deal with that. What we cannot deal with is that you cannot really `patch_status` to update particular condition entry.

none of the original patch types even work (strategic might have, but not supported on crds). so you need at least server side apply to even use conditions.

SKIP DUE TO https://github.com/clux/kube-rs/issues/43 FIXED IN SS APPLY?


---
### WatchEvent::Bookmark

```json
{"type":"BOOKMARK","object":{ \
    "kind":"Pod","apiVersion":"v1", \
    "metadata":{"resourceVersion":"3845","creationTimestamp":null},\
    "spec":{"containers":null},"status":{}}}
```

[k8s-openapi#70](https://github.com/Arnavion/k8s-openapi/issues/70#issuecomment-651224856)

notes:
- no spec, no name, kind Pod.
- so that actually validates `metadata.name` being optional (even if we didn't have a generatename mechanism)
- however, it sets containers to `null`, which is actually against spec so openapi dependents can't actually parse this (need to raise this upstream)
