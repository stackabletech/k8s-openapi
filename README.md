[![Build Status](https://dev.azure.com/arnavion/k8s-openapi/_apis/build/status/Arnavion.k8s-openapi?branchName=master)](https://dev.azure.com/arnavion/k8s-openapi/_build/latest?definitionId=1)

This crate is a Rust Kubernetes API client. It contains bindings for the resources and operations in the Kubernetes client API, auto-generated from the OpenAPI spec.

[crates.io](https://crates.io/crates/k8s-openapi)

[Documentation](https://docs.rs/k8s-openapi)


This crate is *not* generated using Swagger directly, as clients generated by [the common client generator](https://github.com/kubernetes-client/gen) are. This gives this crate a few important advantages.


### Works around bugs in the upstream OpenAPI spec

The upstream OpenAPI spec is not written by hand; it's itself generated from the API server's Go code. As such, the spec makes mistakes when trying to convert the representations of Go types to OpenAPI, such as incorrectly representing the `JSON` type used in CRD validation, and incorrectly representing the type of objects inside a `WatchEvent`. A Swagger-generated client inherits all these mistakes, and it is hard for such a client to fix them in "post".

Since this crate does not use Swagger directly, it is able to work around these mistakes and emit correct bindings. See the list of fixes [here](https://github.com/Arnavion/k8s-openapi/blob/master/src/fixups.rs) and the breakdown of fixes applied to each Kubernetes version [here.](https://github.com/Arnavion/k8s-openapi/blob/master/src/supported_version.rs)


### Better code organization, closer to the Go API

Swagger-generated clients tend to place all the API operations in massive top-level modules. For example, the Python client contains a single [CoreV1Api class](https://github.com/kubernetes-client/python/blob/master/kubernetes/client/apis/core_v1_api.py) with a couple of hundred methods, one for each `core/v1` API operation like `list_namespaced_pod`.

This crate instead associates these functions with the corresponding resource type. The `list_namespaced_pod` function is accessed as `Pod::list_namespaced_pod`, where `Pod` is the resource type for pods. This is similar to the Go API's [PodInterface::List](https://godoc.org/k8s.io/client-go/kubernetes/typed/core/v1#PodInterface)

Since all types are under the `io.k8s` namespace, this crate also removes those two components from the module path. The end result is that the path to `Pod` is `k8s_openapi::api::core::v1::Pod`, similar to the Go path `k8s.io/api/core/v1.Pod`.

Furthermore, the OpenAPI spec contains many redundant type aliases under the `io.k8s.kubernetes.pkg` namespace, for backwards-compatibility with v1.7. Swagger-generated clients usually emit these in some form, whereas this crate just ignores them. This reduces compilation time greatly.


### Better handling of optional parameters, for a more Rust-like and ergonomic API

Almost every API operation has optional parameters. For example, v1.13's `list_namespaced_pod` API has one required parameter (the namespace) and nine optional parameters.

The clients of other languages use language features to allow the caller to not specify all these parameters when invoking the function. The Python client's functions parse optional parameters from `**kwargs`. The C# client's functions assign default values to these parameters in the function definition.

Since Rust does not have such a feature, Swagger-generated Rust clients use `Option<>` parameters to represent optional parameters. This ends up requiring callers to pass in a lot of `None` parameters just to satisfy the compiler. Invoking the `list_namespaced_pod` of a Swagger-generated client would look like:

```rust
// List all pods in the kube-system namespace
list_namespaced_pod("kube-system", None, None, None, None, None, None, None, None, None);

// List all pods in the kube-system namespace with label foo=bar
list_namespaced_pod("kube-system", None, None, None, /* label_selector */ Some("foo=bar"), None, None, None, None, None);
```

Apart from being hard to read, you could easily make a typo and pass in `Some("foo=bar")` for one of the four other optional String parameters without any errors from the compiler.

This crate moves all optional parameters to separate structs, one for each API. Each of these structs implements `Default` and the names of the fields match the function parameter names, so that the above calls look like:

```rust
// List all pods in the kube-system namespace
list_namespaced_pod("kube-system", Default::default());

// List all pods in the kube-system namespace with label foo=bar
list_namespaced_pod("kube-system", ListNamespacedPodOptional { label_selector: Some("foo=bar"), ..Default::default());
```

The second example uses struct update syntax to explicitly set one field of the struct and `Default` the rest.


### Not restricted to a single HTTP client implementation, and works with both synchronous and asynchronous HTTP clients

Swagger-generated clients have to choose between providing a synchronous or asynchronous API, and have to choose what kind of HTTP client they want to use internally (`hyper::Client`, `reqwest::Client`, `reqwest::r#async::Client`, etc). If you want to use a different HTTP client, you cannot use the crate.

This crate is instead based on the [sans-io approach](https://sans-io.readthedocs.io/) popularized by Python for network protocols and applications.

For example, the `list_namespaced_pod` does not return `Result<ListNamespacedPodResponse>` or `impl Future<Item = ListNamespacedPodResponse>`. It returns an `http::Request<Vec<u8>>` with the URL path, query string and request body filled out. You are free to execute this `http::Request` using any HTTP client you want to use.

The `ListNamespacedPodResponse` type has `try_from_parts(http::StatusCode, &[u8]) -> Result<(Self, usize), crate::ResponseError>` function which knows how to parse a combination of HTTP status code and response bytes into the appropriate result. No matter how you executed the request, you would have a status code and some response bytes. The function returns either a successful response, or an error if the response could not be parsed or if more response bytes are needed (ie you need to call the function again after reading more bytes from your HTTP response).

There is also a top-level `ResponseBody` type that contains its own internal growable byte buffer, if you don't want to manage a byte buffer yourself. See the crate docs for details.


### Supports more versions of Kubernetes

Official clients tend to support only the three latest versions of Kubernetes. This crate supports a few more.

As mentioned above, the upstream OpenAPI spec contains mistakes. When upstream fixes these mistakes, it usually does not backport them to older versions (not even to *supported* older versions). This crate does backport those fixes if they're applicable.


# License

```
k8s-openapi

https://github.com/Arnavion/k8s-openapi

Copyright 2018 Arnav Singh

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

The OpenAPI spec that these bindings are generated from is sourced from the
Kubernetes repository https://github.com/kubernetes/kubernetes which also uses
the Apache-2.0 license.
