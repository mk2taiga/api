# Istio API Guidelines

This page defines the design guidelines for Istio APIs. They apply to
[Kubernetes Custom Resource Definitions](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
and all [proto files](https://developers.google.com/protocol-buffers/) that are used to
configure Istio components through the [Mesh Configuration Protocol(MCP)](https://docs.google.com/document/d/1o2-V4TLJ8fJACXdlsnxKxDv2Luryo48bAhR8ShxE5-k/edit).

Since MCP is based on _proto3_ and _gRPC_, we will use
Google's [API Design Guide](https://cloud.google.com/apis/design) as
the baseline for protos. Because Envoy APIs also uses the same baseline, the
commonality across Envoy, Istio, proto3 and gRPC will greatly help
developer experience in the long term.

In addition to Google's guide, the following conventions should be
followed for Istio APIs.

## Contents

- [Proto Guidelines](#proto-guidelines)
    - [Style](#style)
    - [Basic Proto Versioning](#basic-proto-ersioning)
- [Validation Guidelines](#validation-guidelines)
- [CRD Guidelines](#crd-guidelines)
    - [Style](#crd-style)
    - [Basic CRD Versioning](#basic-crd-versioning)

## Proto Guidelines

This section captures guidelines that apply to the proto form of the configuration resources.

### Style

#### Placement

- **Do** place new API protos under ```istio.io/api/<area>/<version>``` folder.
- **Prefer** complete words for file names.

    ```
    index.proto // Not idx.proto!
    ```

#### Package Names

- **Do** use `lowercase` without any `_`.
- **Do** use singular words
- **Do** use the name pattern ```istio.<area>.<version>```.

    ```proto
    package istio.networking.v1alpha3;
    ```

#### Message/Enum/Method Names

- **Do not** use embedded acronyms. See [#364](https://github.com/istio/api/issues/364) for details.

    ```proto
    message HttpRequest {/*...*/}  // Not HTTPRequest!

    rpc DoHttpRequest(/*...*/)     // Not DoHTTPRequest!

    enum HttpStatusCodes {/*...*/} // Not HTTPStatusCodes!
    ```

#### Messages

- **Do** use ```CamelCase``` for message names.

    ```proto
    message MyMessage {...}
    ```

#### Fields

- **Do** use ```lowercase_with_underscore``` for field names:

    ```proto
    string display_name = 1;
    ```

- **Do** use plural names for repeated fields:

    ```proto
    repeated rule rules = 2;
    ```

- **Do not** use postpositive adjectives in names.

    ```proto
    repeated Items collected_items = 3; // Not items_collected!
    ```

#### Enums

- **Do** use ```CamelCase``` for  types names.

    ```proto
    enum Types {/*...*/}
    ```

- **Do** use `UPPERCASE_WITH_UNDERSCORE` for enum names:

    ```proto
    enum Types {INT_TYPE = 1;}
    ```

- **Do** have an enum entry for value ```0```.

    - When a new field with an enum type is introduced to a proto, when reading
    the older versions of the data, it will be defaulted to the ```0``` value.

- **Do** name the ```0``` value either as ```<ENUMTYPE>_UNDEFINED``` or use a sane,
well-known value that will be considered as default.

    ```proto
    enum Types { TYPE_UNDEFINED = 0;}
    ```

### Basic Proto Versioning

Protocol Buffers have well-defined semantics for coping with version changes. In essense, the unknown fields
get ignored, and additive, and non-breaking changes are acceptable.

In addition to the standard proto versioning semantics, Istio tooling imposes its own restrictions, as CRD
to proto conversion system in Istio depends on names in certain situations.

The following rules captures the basic rules that should be followed when making changes to Istio config
protos.

- **Do not** change field numbers.

    - Proto depends on field numbers to find fields.

        ```proto
        // Field number has changed from 1 to 2!

        // string field = 1; // Deleted
        string field = 2;
        ```

- **Do not** rename fields.

    - Our tooling automatically maps YAML fields to proto fields.

        ```proto
        // Field name has changed!

        // string old_field = 1;
        string new_field = 1;
        ```

- **Do not** change cardinality of fields.

    ```proto
    // Field cardinality has changed!

    // string should_have_been_plural = 1;
    repeated string should_have_been_plural = 1;
    ```

- **Do not** rename top-level protos that map to CRD config types.

    - Istio tooling depends on the name of the top-level protos to map CRDs to the matching proto counterparts.

        ```proto
        // Top-level proto name has changed!

        // message Rule {
        message PolicyRule {
            // ...
        }
        ```

- **Avoid** changing/renaming field types.

    - If the field types changes, the new type **must** be structurally equivalent to the old.

        ```proto
        // Field type changed from Boo to Zoo!
        // Boo and Zoo must be structurally equivalent!
        message Foo {
           // Boo boo = 1;
           Zoo boo = 2;
        }
        ```

- **Do not** rename enum names, or change values.

    - Istio tooling depends on names to convert enums from CRD form to proto.

        ```proto
        enum Types {
           // Enum name has changed!
           // Foo = 1;
           Bar = 1;

           // Enum value has changed!
           // Baz = 2;
           Baz = 3;
        }
        ```

- **Do not** remove fields.

    - This is backwards compatible for protobuf, but not for CRDs which have strict validation preventing unknown fields.

- **Do not** make validation stricter than in previous versions.

    - This applies to OpenAPI schema validation, validation webhooks, or any similar validation that would reject and API.

    - Previously valid APIs must continue to remain valid in future upgrades; a change to validation is just as impactful as
      removal of a field.

    - For example, changing a `string` value to have a max length of X characters would break users with
      configurations beyond X characters upon upgrade, and would not be permitted.

    - Loosening validation is permitted. As a result, it is recommended to err on the side of stricter validation.

## Validation Guidelines

All types should have as strict validation specified on it as possible to rule out invalid states.
These are ultimately compiled to Kubernetes CustomResourceDefinitions, which use OpenAPI validation with some Kubernetes extras.
This is handled by our own custom [protoc-gen-crd](https://github.com/istio/tools/tree/master/cmd/protoc-gen-crd) which compiles our
protobuf definitions down to CRDs.

There are a few types of validations:
* Automatic ones, based on the protobuf type. For example, a UInt32Value automatically has a validation to check the number between `0` and `MaxUint32`
* Protobuf `field_behavior`. Currently only `[(google.api.field_behavior) = REQUIRED]` is implemented.
* Comment driven validations (see below).

Most validation is driven by comments on fields and messages.
All validations in [KubeBuilder](https://book.kubebuilder.io/reference/markers/crd-validation) are supported, as well as some extras:

- `+protoc-gen-crd:map-value-validation`: apply the validation to each *value* in a map.
  Note it's not possible to apply validations to each key. You can, however, validate the entire map together with a CEL rule.
- `+protoc-gen-crd:list-value-validation`: apply the validation to each value in a list.
- `+protoc-gen-crd:duration-validation:none`: exclude the default requirement that a duration field is non-zero.
- `+protoc-gen-crd:validation:XIntOrString`: marks a field as accepting integers or strings.
- `+protoc-gen-crd:validation:IgnoreSubValidation`: if referencing a message in a field, and that message has some validation on it already, exclude the listed validations.
  This is uncommon, but can be used when referencing a message in a certain context has different rules than others.

The most common validations are:
- Sizes: `MaxLength` (strings), `MaxItems` (lists), `MaxProperties` (maps)
- Regex: `Pattern`
- CEL: `XValidation`

### CEL

[CEL](https://cel.dev/) is a small language that allows us to write expressions to represent validation logic.
This comes with a lot of quirks!

Useful tools and references:
* [CEL playground](https://playcel.undistro.io/) allows an easy way to run CEL expressions against some types.
* [Kubernetes CEL docs](https://kubernetes.io/docs/reference/using-api/cel/).
* [CEL language definition](https://github.com/google/cel-spec/blob/master/doc/langdef.md).

The biggest challenge with CEL is the complexity limit imposed by Kubernetes.
This estimates the cost to run the function, and rejects it if it is too high.
This takes into account the cost of a function and the cost of *potential* inputs.
This makes it, typically, required to put maximum size bounds on items.

Kubernetes changes version-to-version on how it estimates cost (usually getting more lenient) and what functions are available.
We want to target the oldest version for compatibility purposes.
Our tests do not currently cover this (a prototype of doing so can be found [here](https://github.com/istio/api/pull/3275)).
A list of what features are in which versions can be found [here](https://kubernetes.io/docs/reference/using-api/cel/#cel-options-language-features-and-libraries).

Istio has some custom macros that are expanded at compile time, driven by the [celpp](https://github.com/howardjohn/celpp) package.
This extends CEL with these capabilities:
* **default**. Usage: `default(self.x, 'DEF')`.
* **oneof**. Usage: `oneof(self.x, self.y, self.z)`. This checks that 0 or 1 of these fields is set.
* **index**. Usage: `self.index({}, x, z, b)`. This does `self.x.z.b` and returns `{}` if any of these is not set.

Unlike typical Go usage, CEL does not have a concept of zero values for unset fields.
As a result, an optional field needs special care.
Do not write `self.fruit == 'apple'`, for instance, write `default(self.fruit, '') == 'apple'.

### Testing

As validation logic is really easy to get wrong, it's useful to write tests.
This is done by adding YAML files under `tests/testdata`.
Each type has a `valid` and `invalid` file to do positive and negative cases.

Aside from explicitly testing these, these also form the seed corpus for fuzzing when these are pulled into `istio/istio`.
This fuzz testing verifies the CRD validation has the same result as the webhook (Golang) validation code.
Currently, this mostly serves to ensure we do not make something overly strict.
In the future, it may show us that its safe to disable the webhook entirely, if CRD validation can cover the full validation surface.

## CRD Guidelines

### CRD Style

- **Do** use the name pattern ```<area>.istio.io``` for API group names.
- **Do** use the version from the proto package as the API version.
- **Do** use the top-level proto Message name as the ```kind``` of the CRD.

```yaml
# A custom resource that describes a Gateway resource
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
# ...
```

Matches to:

```proto
package istio.networking.v1alpha3;
message Gateway {
    // ...
}
```

### Basic CRD Versioning

Istio APIs should use a simple versioning strategy based on
major versions and releases, such as `v1alpha`, `v2beta`, or
`v3`. Due to current limitations in Istio's CRD versioning, namely
a lack of a conversion webhook, the schema of all versions of an API
must be strictly identical.

New APIs progress through feature stages according to our [policy](https://istio.io/latest/docs/releases/feature-stages/) before achieving stability, `v1`. When adding a new API, follow these steps:
- Add `// +cue-gen:<Replace with API Name>:releaseChannel:extended` tag to the proto message definition of the API.
- Update the stable validation policy to exclude the new API in the [base chart](https://github.com/istio/istio/blob/199f76a601fc4520b675169d4b53503edfaa34e3/manifests/charts/base/templates/validatingadmissionpolicy.yaml#L31) and [istio-discovery chart](https://github.com/istio/istio/blob/199f76a601fc4520b675169d4b53503edfaa34e3/manifests/charts/istio-control/istio-discovery/templates/validatingadmissionpolicy.yaml#L37) manifests.

Similarly, new API fields added to a stable `v1` API independently progress through feature stages based on our [policy](https://istio.io/latest/docs/releases/feature-stages/) before achieving stability. When adding a new API field to a stable `v1` API, follow these steps:
- Add `// +cue-gen:<Replace with API Name>:releaseChannel:extended` tag to the proto field definition.
- Update the stable validation policy to exclude the new field in the [base chart](https://github.com/istio/istio/blob/199f76a601fc4520b675169d4b53503edfaa34e3/manifests/charts/base/templates/validatingadmissionpolicy.yaml#L31) and [istio-discovery chart](https://github.com/istio/istio/blob/199f76a601fc4520b675169d4b53503edfaa34e3/manifests/charts/istio-control/istio-discovery/templates/validatingadmissionpolicy.yaml#L37) manifests.

Deprecating a feature in an API release is allowed by following
the applicable deprecation process. The reason to allow
deprecation of individual features in a release is that it is
significantly cheaper and simpler for everyone involved. In
practice, it works out much better than deprecating an entire
API version.

## Exceptions

Many of the guidelines above are related to limiting backward incompatible changes.
These guidelines apply only between released versions of Istio (including patches
and minor releases). This means that if a commit is merged into the `master` branch,
breaking changes can still be made to it (such as removal, renaming, etc) up until
it has been officially released.

While violating the above guidelines is generally disallowed, exceptions may be made
on a case-by-case basis.
