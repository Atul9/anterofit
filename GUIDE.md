Anterofit User's Guide
======================

Anterofit makes it easy to abstract over REST APIs and asynchronous requests.

The core of Anterofit's abstraction power lies in its macro-based
generation of service traits, eliminating the noisy boilerplate
involved in creating and issuing HTTP requests. The
focal point of this power lies in the `service!{}` macro.

See the [README](README.md) for setting up dependencies and choosing a serialization framework.

Creating Service Traits with `service!{}`
-----------------------------------------

`service!{}` simply takes the definition of a trait item, with its method bodies in a particular format,
and generates an object-safe implementation of the service trait for the `Adapter` type.

```rust
service! {
    /// Trait wrapping `myservice.com` API.
    pub trait MyService {
        /// Get the version of this API.
        fn api_version(&self) -> String {
            GET("/version")
        }

        /// Register a user with the API.
        fn register(&self, username: &str, password: &str) {
            POST("/register");
            fields! {
                username, password
            }
        }
    }
}
```

#### Service Trait Method Overview

Service trait methods always take `&self` as the first parameter; this is purely
an implementation detail. Method parameters are passed to the implementation, unchanged.
They can be borrowed or owned, but there is some restriction on the usage
of borrowed parameters.

The return type of a trait method can be any type that implements the correct deserialization trait
for the serialization framework you're using:

* `rustc-serialize`: `Decodable`, derived with `#[derive(RustcDecodable)]`

* Serde: `Deserialize`, derived with `#[derive(Deserialize)]` via either 
[`serde_codegen`](https://crates.io/crates/serde_codegen) (build script) or 
[`serde_derive`](https://crates.io/crates/serde_derive) (procedural macro).

The return type can also be omitted. Just like in regular Rust, it is implied to be `()`.

The body of a trait method is syntactically the same
as any (non-empty) Rust function: zero or more semicolon-terminated statements/expressions 
followed by an unterminated expression. However, there are a couple of major differences:

#### HTTP Verb and URL

The first expression, which is always required, is structured like a function call, where the identifier 
outside is an HTTP verb and the inside is the URL string and any optional formatting arguments, in the vein
of `format!()` or `println!()`. This allows parameters to be interpolated into the URL. 
The most common HTTP verbs are supported: `GET POST PUT PATCH DELETE`

```
// If `id` is some parameter that implements `Display`
GET("/version")
GET("/posts/{}", id)
POST("/posts/update/{}", id)
DELETE("/posts/{id}", id=id)
```

Notice that the paths in these declarations are not assumed to be complete URLs; instead, they will be appended to the 
base URL provided in the adapter. However, if necessary, they *can* be complete URLs, with the base URL being omitted 
during the construction of the adapter.

#### Request Modifiers (query pairs, form fields, etc)

All expressions following the first, if any, are treated as modifiers to the request. 
Syntactically, any expression is allowed, but arbitrary expressions will likely not typecheck due to some
implementation details of the `service!{}` macro. Instead, you are expected to use the other macros provided 
by Anterofit to modify the request.

See the [`Macros` header in the crate docs][doc-macros] for more information.

* To add query parameters, sometimes called `GET` parameters, use `query!{}`
 
* To add form fields, sometimes called `POST` parameters, use `fields!{}`.
 You can see this being used in the first example.
 
    * To add a file to be uploaded, use `path!()` (takes anything convertible to `PathBuf`) as a field value.
    
    * To add a stream to be uploaded (can be any generic `Read` impl), use `stream!()` as a field value.
    
* To set the request body, use the `body!()` macro. You would use this if your REST API is expecting parameters 
passed as, e.g. JSON, instead of in an HTTP form; see the [Serialization](#serialization) header for more information.

* To set the request body as a series of key-value pairs, use `body_map!()`. This behaves as if you passed
a `HashMap` or `BTreeMap` of the key-value pairs to `body!()`, but does not require the keys to implement 
any trait except `std::fmt::Display` (thus, keys are not deduplicated or reordered--the server is expected to handle
it); values are, of course, expected to implement the serialization trait from the serialization framework you're using.

* To apply arbitrary mutations or transformations to the request builder, use `with_builder!()` or `map_builder!()`, 
respectively.

For more advanced usage, you can use bare closure expressions that take `RequestBuilder` and return 
`Result<RequestBuilder, anterofit::Error>`. See `RequestBuilder::apply()`, which is used as a type hint
so that no type annotations are required on the closures (very convenient, by the way). All the aforementioned
macros wrap this mechanism.

[doc-macros]: http://docs.rs/anterofit#macros

Serialization
-------------

Anterofit supports both serialization of request bodies, and deserialization of response bodies.