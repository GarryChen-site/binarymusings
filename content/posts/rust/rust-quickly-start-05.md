---
title: "Build Image Server in Rust: Thumbor Clone with 300 Lines"

description: "Create a powerful image processing server in Rust like Thumbor. Dynamic resize, crop, watermark, and filters with caching in just 300 lines of code."

summary: "Build a production-ready image processing server by recreating Thumbor in Rust with only 300 lines of code. Learn to implement dynamic image transformations including resizing, cropping, watermarking, and filters. Discover protobuf for extensible APIs, LRU caching for performance, async web server with Axum, and trait-based architecture for scalability. A comprehensive tutorial demonstrating Rust's power for real-world web services."

date: 2025-08-03
series: ["Rust"]
weight: 1
tags: ["rust", "image-processing", "web-server", "thumbor", "rust-web-service"]
author: ["Garry Chen"]
cover:
  image: images/rust-05-00.webp
  hiddenInList: true
  caption: "image server in rust"

---


In the previous lesson, we wrote the HTTPie tool using just over 100 lines of code. If you're eager for more, today we'll create another practical example to explore more of Rust's capabilities.

Again, if you don't fully understand the code, don't worry. Just follow along line by line, run the code, and feel the difference between Rust and the languages you're familiar with. Observe the coding style and get your code running—understanding every detail isn't necessary right now.

Today's example is a common requirement we encounter in our work: building a web server to provide certain services. Similar to the previous lesson with HTTPie, we'll reimplement an existing open-source tool in Rust. Today, we'll tackle a slightly larger project: building an image server similar to [Thumbor](https://github.com/thumbor/thumbor).

## Thumbor

Thumbor is a well-known image server in the Python ecosystem, widely used in scenarios requiring dynamic image resizing.

It can dynamically crop and resize images through a simple HTTP interface and supports file storage and other auxiliary features.


```plaintext
http://<thumbor-server>/300x200/smart/thumbor.readthedocs.io/en/latest/_images/logo-thumbor.png
```
In this example, Thumbor crops the image using smart crop and resizes it to 300x200. Users accessing this URL get a 300x200 thumbnail.

Today, we'll implement **the core functionality: dynamic image transformation**. Consider how you would design this service using the language you're most familiar with, what libraries you'd use, and how many lines of code it might take. Now, think about how many lines of code it would take with Rust.

With your thoughts in mind, let's start building this tool in Rust! Our goal is to meet our requirements with about 200 lines of code.

## Design Analysis

Since this is an image transformation, we must support various transformation functions like resizing, cropping, watermarking, and even applying filters. However, the challenge in an image transformation service lies in the interface design—creating a simple, easy-to-use interface that allows for future extensions.

Why is this important? Imagine if one day a product manager asks you to support a filter effect for old photos in a service initially designed only for creating thumbnails. How would you handle it?

Thumbor's solution is to place the processing methods in a specific format and order in the URL path, omitting unused methods:

```plaintext
/hmac/trim/AxB:CxD/(adaptative-)(full-)fit-in/-Ex-F/HALIGN/VALIGN/smart/filters:FILTERNAME(ARGUMENT):FILTERNAME(ARGUMENT)/*IMAGE-URI*
```
However, this approach isn't easily extendable, isn't convenient to parse, and doesn't handle ordered operations well. For example, you might want to apply a filter before adding a watermark to one image, but apply the watermark before the filter to another image.

Moreover, adding more parameters in the future could lead to conflicts with existing parameters or cause breaking changes. As developers, we should never underestimate the creative and often unexpected requests of product managers.

So, when designing this project, **we need a simpler, more extensible way to describe a series of ordered image operations**, such as resizing, then adding a watermark to the resized result, and finally applying a filter.

In code, these ordered operations can be represented by a list, where each operation is an enum, like this:

```rust
// Parsed image processing parameters
struct ImageSpec {
    specs: Vec<Spec>
}

// Each parameter represents a supported processing method
enum Spec {
    Resize(Resize),
    Crop(Crop),
    ...
}

// Process image resizing
struct Resize {
    width: u32,
    height: u32
}
```

Now that we have the necessary data structures, **how do we design an interface that any client can use and reflect in the URL to parse into our data structures**?

Using a query string? Although feasible, it can become unwieldy when image processing steps are complex, such as needing seven or eight transformations for an image, resulting in a very long query string.

My approach is to use protobuf. Protobuf can describe data structures and has support in almost every language. Once an image spec is generated with protobuf, it can be serialized into a byte stream. However, a byte stream can't be placed in a URL—so what do we do? We can encode it using base64!

Following this approach, let's write the protobuf message definition describing the image spec:

```protobuf
message ImageSpec {
  repeated Spec specs = 1;
}

message Spec {
  oneof data {
    Resize resize = 1;
    Crop crop = 2;
    ...
  }
}
```

With this, we can embed a base64-encoded string generated by protobuf into the URL to provide extensible image processing parameters. The processed URL looks like this:

```plaintext
http://localhost:3000/image/CgoKCAjYBBCgBiADCgY6BAgUEBQKBDICCAN/<encoded origin url>
```

The string `CgoKCAjYBBCgBiADCgY6BAgUEBQKBDICCAN` describes the image processing steps: resizing, adding a watermark to the resized result, and applying a filter. This can be implemented with the following code:

```rust
fn print_test_url(url: &str) {
    use std::borrow::Borrow;
    let spec1 = Spec::new_resize(600, 800, resize::SampleFilter::CatmullRom);
    let spec2 = Spec::new_watermark(20, 20);
    let spec3 = Spec::new_filter(filter::Filter::Marine);
    let image_spec = ImageSpec::new(vec![spec1, spec2, spec3]);
    let s: String = image_spec.borrow().into();
    let test_image = percent_encode(url.as_bytes(), NON_ALPHANUMERIC).to_string();
    println!("test url: http://localhost:3000/image/{}/{}", s, test_image);
}
```

Using protobuf has the advantage of producing a compact serialized result, and any language supporting protobuf can generate or parse this interface.

With the interface decided, the next step is to create an HTTP server to provide this interface. In the HTTP server's `/image` route handler, we need to obtain the original image from the URL, process it according to the image spec, and return the processed byte stream to the user.

An obvious optimization is to **provide an LRU (Least Recently Used) cache for the process of obtaining the original image**, as accessing external networks is the slowest and most unpredictable step in the entire path.

![thumbor-arch](images/rust-05-01.webp)

After this analysis, doesn't Thumbor seem not so complex? But you might still wonder: can we really complete all this work in 200 lines of code? Let's start writing and count the lines after we finish.


## Protobuf Definition and Compilation

Let's start by creating a new project with `cargo new thumbor`, then add the following dependencies to the `Cargo.toml` file:

```toml
[dependencies]
axum = "0.2" # Web server
anyhow = "1" # Error handling
base64 = "0.13" # Base64 encoding/decoding
bytes = "1" # Byte stream handling
image = "0.23" # Image processing
lazy_static = "1" # Convenient static variable initialization via macros
lru = "0.6" # LRU cache
percent-encoding = "2" # URL encoding/decoding
photon-rs = "0.3" # Image effects
prost = "0.8" # Protobuf handling
reqwest = "0.11" # HTTP client
serde = { version = "1", features = ["derive"] } # Data serialization/deserialization
tokio = { version = "1", features = ["full"] } # Asynchronous processing
tower = { version = "0.4", features = ["util", "timeout", "load-shed", "limit"] } # Service handling and middleware
tower-http = { version = "0.1", features = ["add-extension", "compression-full", "trace"] } # HTTP middleware
tracing = "0.1" # Logging and tracing
tracing-subscriber = "0.2" # Logging and tracing

[build-dependencies]
prost-build = "0.8" # Protobuf compilation
```

Create a `abi.proto` file in the root directory of your project and define the data structures used by our image processing services:

```proto
syntax = "proto3";

package abi; // This name will be used for the compilation result, prost will generate: abi.rs

// An ImageSpec is an ordered array, the server processes according to the spec sequence
message ImageSpec {
  repeated Spec specs = 1;
}

// Processing image resizing
message Resize {
  uint32 width = 1;
  uint32 height = 2;

  enum ResizeType {
    NORMAL = 0;
    SEAM_CARVE = 1;
  }

  ResizeType rtype = 3;

  enum SampleFilter {
    UNDEFINED = 0;
    NEAREST = 1;
    TRIANGLE = 2;
    CATMULL_ROM = 3;
    GAUSSIAN = 4;
    LANCZOS3 = 5;
  }

  SampleFilter filter = 4;
}

// Processing image cropping
message Crop {
  uint32 x1 = 1;
  uint32 y1 = 2;
  uint32 x2 = 3;
  uint32 y2 = 4;
}

// Processing horizontal flip
message Fliph {}
// Processing vertical flip
message Flipv {}
// Processing contrast
message Contrast {
  float contrast = 1;
}
// Processing filter
message Filter {
  enum Filter {
    UNSPECIFIED = 0;
    OCEANIC = 1;
    ISLANDS = 2;
    MARINE = 3;
    // more: https://docs.rs/photon-rs/0.3.1/photon_rs/filters/fn.filter.html
  }
  Filter filter = 1;
}

// Processing watermark
message Watermark {
  uint32 x = 1;
  uint32 y = 2;
}

// A spec can contain one of the above processing methods
message Spec {
  oneof data {
    Resize resize = 1;
    Crop crop = 2;
    Flipv flipv = 3;
    Fliph fliph = 4;
    Contrast contrast = 5;
    Filter filter = 6;
    Watermark watermark = 7;
  }
}
```

This defines the image processing services we support and allows for easy expansion to support more operations in the future.

Protobuf is a backward-compatible tool, so as the server supports more features, it can still be compatible with older clients. In Rust, we use `prost` to use and compile protobuf. Create a `build.rs` file in the root directory and add the following code:

```rust
fn main() {
    prost_build::Config::new()
        .out_dir("src/pb")
        .compile_protos(&["abi.proto"], &["."])
        .unwrap();
}
```

`build.rs` can handle additional compilation steps when building the cargo project. Here we use `prost_build` to compile `abi.proto` into the `src/pb` directory.

The directory does not exist yet, so you need to create it with `mkdir src/pb`. Run `cargo build`, and you will see a `abi.rs` file generated in `src/pb`. This file contains the Rust data structures converted from protobuf messages. For now, just treat them as regular data structures.

Next, create `src/pb/mod.rs` to **declare all the code in a directory through `mod.rs`**. In this file, we import `abi.rs` and write some helper functions to easily convert `ImageSpec` to and from a string.

Additionally, we wrote a test to ensure functionality correctness. You can run `cargo test` to test it. Remember to add `mod pb;` in `main.rs` to import this module.

```rust
use base64::{decode_config, encode_config, URL_SAFE_NO_PAD};
use photon_rs::transform::SamplingFilter;
use prost::Message;
use std::convert::TryFrom;

mod abi; // Declare abi.rs
pub use abi::*;

impl ImageSpec {
    pub fn new(specs: Vec<Spec>) -> Self {
        Self { specs }
    }
}

// Convert ImageSpec to a string
impl From<&ImageSpec> for String {
    fn from(image_spec: &ImageSpec) -> Self {
        let data = image_spec.encode_to_vec();
        encode_config(data, URL_SAFE_NO_PAD)
    }
}

// Create ImageSpec from a string, e.g., s.parse().unwrap()
impl TryFrom<&str> for ImageSpec {
    type Error = anyhow::Error;

    fn try_from(value: &str) -> Result<Self, Self::Error> {
        let data = decode_config(value, URL_SAFE_NO_PAD)?;
        Ok(ImageSpec::decode(&data[..])?)
    }
}

// Helper function, photon_rs methods need strings
impl filter::Filter {
    pub fn to_str(&self) -> Option<&'static str> {
        match self {
            filter::Filter::Unspecified => None,
            filter::Filter::Oceanic => Some("oceanic"),
            filter::Filter::Islands => Some("islands"),
            filter::Filter::Marine => Some("marine"),
        }
    }
}

// Convert between our SampleFilter and photon_rs's SamplingFilter
impl From<resize::SampleFilter> for SamplingFilter {
    fn from(v: resize::SampleFilter) -> Self {
        match v {
            resize::SampleFilter::Undefined => SamplingFilter::Nearest,
            resize::SampleFilter::Nearest => SamplingFilter::Nearest,
            resize::SampleFilter::Triangle => SamplingFilter::Triangle,
            resize::SampleFilter::CatmullRom => SamplingFilter::CatmullRom,
            resize::SampleFilter::Gaussian => SamplingFilter::Gaussian,
            resize::SampleFilter::Lanczos3 => SamplingFilter::Lanczos3,
        }
    }
}

// Provide helper functions to simplify creating a spec
impl Spec {
    pub fn new_resize_seam_carve(width: u32, height: u32) -> Self {
        Self {
            data: Some(spec::Data::Resize(Resize {
                width,
                height,
                rtype: resize::ResizeType::SeamCarve as i32,
                filter: resize::SampleFilter::Undefined as i32,
            })),
        }
    }

    pub fn new_resize(width: u32, height: u32, filter: resize::SampleFilter) -> Self {
        Self {
            data: Some(spec::Data::Resize(Resize {
                width,
                height,
                rtype: resize::ResizeType::Normal as i32,
                filter: filter as i32,
            })),
        }
    }

    pub fn new_filter(filter: filter::Filter) -> Self {
        Self {
            data: Some(spec::Data::Filter(Filter {
                filter: filter as i32,
            })),
        }
    }

    pub fn new_watermark(x: u32, y: u32) -> Self {
        Self {
            data: Some(spec::Data::Watermark(Watermark { x, y })),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::borrow::Borrow;
    use std::convert::TryInto;

    #[test]
    fn encoded_spec_could_be_decoded() {
        let spec1 = Spec::new_resize(600, 600, resize::SampleFilter::CatmullRom);
        let spec2 = Spec::new_filter(filter::Filter::Marine);
        let image_spec = ImageSpec::new(vec![spec1, spec2]);
        let s: String = image_spec.borrow().into();
        assert_eq!(image_spec, s.as_str().try_into().unwrap());
    }
}
```

## Introducing the HTTP Server

Having handled protobuf-related content, let's move on to the HTTP service flow. The Rust community has many high-performance web servers, such as [actix-web](https://github.com/actix/actix-web), [rocket](https://github.com/rwf2/Rocket), [warp](https://github.com/seanmonstar/warp), and [axum](https://github.com/tokio-rs/axum).

Based on the Axum documentation, we can construct the following code:

```rust
use axum::{extract::Path, handler::get, http::StatusCode, Router};
use percent_encoding::percent_decode_str;
use serde::Deserialize;
use std::convert::TryInto;

// Include the generated protobuf code; we won't worry about these for now
mod pb;

use pb::*;

// Use serde to deserialize parameters; Axum will automatically recognize and parse them
#[derive(Deserialize)]
struct Params {
    spec: String,
    url: String,
}

#[tokio::main]
async fn main() {
    // Initialize tracing
    tracing_subscriber::fmt::init();

    // Build the router
    let app = Router::new()
        // `GET /image` will execute the generate function and pass the spec and url parameters
        .route("/image/:spec/:url", get(generate));

    // Run the web server
    let addr = "127.0.0.1:3000".parse().unwrap();
    tracing::debug!("Listening on {}", addr);
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

// For now, we'll just parse the parameters
async fn generate(Path(Params { spec, url }): Path<Params>) -> Result<String, StatusCode> {
    let url = percent_decode_str(&url).decode_utf8_lossy();
    let spec: ImageSpec = spec
        .as_str()
        .try_into()
        .map_err(|_| StatusCode::BAD_REQUEST)?;
    Ok(format!("url: {}\n spec: {:#?}", url, spec))
}
```

After adding these to `main.rs`, run the server with `cargo run`.

```sh
httpie get "http://localhost:3000/image/CgoKCAjYBBCgBiADCgY6BAgUEBQKBDICCAM/https%3A%2F%2Fimages%2Epexels%2Ecom%2Fphotos%2F2470905%2Fpexels%2Dphoto%2D2470905%2Ejpeg%3Fauto%3Dcompress%26cs%3Dtinysrgb%26dpr%3D2%26h%3D750%26w%3D1260"
```

```http
HTTP/1.1 200 OK
content-type: "text/plain"
content-length: "901"
date: "Wed, 25 Aug 2021 18:03:50 GMT"

url: https://images.pexels.com/photos/2470905/pexels-photo-2470905.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=750&w=1260
 spec: ImageSpec {
    specs: [
        Spec {
            data: Some(
                Resize(
                    Resize {
                        width: 600,
                        height: 800,
                        rtype: Normal,
                        filter: CatmullRom,
                    },
                ),
            ),
        },
        Spec {
            data: Some(
                Watermark(
                    Watermark {
                        x: 20,
                        y: 20,
                    },
                ),
            ),
        },
        Spec {
            data: Some(
                Filter(
                    Filter {
                        filter: Marine,
                    },
                ),
            ),
        },
    ],
}
```

Wow, our web server interface can already process requests correctly.

Don't worry if some of the syntax looks confusing. We haven't covered ownership, type systems, generics, and other details yet, so it's normal not to understand everything. Just follow my thought process and understand the overall flow.

## Fetching and Caching the Source Image

Next, we'll handle the logic to fetch the source image.

We need to introduce an **LRU cache to cache the source image**. Generally, web frameworks have middleware to handle global state, and Axum is no exception. We can use `AddExtensionLayer` to add a global state, which will be the LRU cache that caches the source images fetched from network requests.

```rust
use anyhow::Result;
use axum::{
    extract::{Extension, Path},
    handler::get,
    http::{HeaderMap, HeaderValue, StatusCode},
    AddExtensionLayer, Router,
};
use bytes::Bytes;
use lru::LruCache;
use percent_encoding::{percent_decode_str, percent_encode, NON_ALPHANUMERIC};
use serde::Deserialize;
use std::{
    collections::hash_map::DefaultHasher,
    convert::TryInto,
    hash::{Hash, Hasher},
    sync::Arc,
};
use tokio::sync::Mutex;
use tower::ServiceBuilder;
use tracing::{info, instrument};

mod pb;

use pb::*;

#[derive(Deserialize)]
struct Params {
    spec: String,
    url: String,
}
type Cache = Arc<Mutex<LruCache<u64, Bytes>>>;

#[tokio::main]
async fn main() {
    // Initialize tracing
    tracing_subscriber::fmt::init();
    let cache: Cache = Arc::new(Mutex::new(LruCache::new(1024)));
    // Build the router
    let app = Router::new()
        // `GET /image/:spec/:url` will execute the generate function
        .route("/image/:spec/:url", get(generate))
        .layer(
            ServiceBuilder::new()
                .layer(AddExtensionLayer::new(cache))
                .into_inner(),
        );

    // Run the web server
    let addr = "127.0.0.1:3000".parse().unwrap();
    print_test_url("https://images.pexels.com/photos/1562477/pexels-photo-1562477.jpeg?auto=compress&cs=tinysrgb&dpr=3&h=750&w=1260");
    info!("Listening on {}", addr);

    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}

async fn generate(
    Path(Params { spec, url }): Path<Params>,
    Extension(cache): Extension<Cache>,
) -> Result<(HeaderMap, Vec<u8>), StatusCode> {
    let spec: ImageSpec = spec
        .as_str()
        .try_into()
        .map_err(|_| StatusCode::BAD_REQUEST)?;

    let url: &str = &percent_decode_str(&url).decode_utf8_lossy();
    let data = retrieve_image(&url, cache)
        .await
        .map_err(|_| StatusCode::BAD_REQUEST)?;

    // TODO: Process the image

    let mut headers = HeaderMap::new();
    headers.insert("content-type", HeaderValue::from_static("image/jpeg"));
    Ok((headers, data.to_vec()))
}

#[instrument(level = "info", skip(cache))]
async fn retrieve_image(url: &str, cache: Cache) -> Result<Bytes> {
    let mut hasher = DefaultHasher::new();
    url.hash(&mut hasher);
    let key = hasher.finish();

    let g = &mut cache.lock().await;
    let data = match g.get(&key) {
        Some(v) => {
            info!("Cache hit: {}", key);
            v.to_owned()
        }
        None => {
            info!("Fetching URL");
            let resp = reqwest::get(url).await?;
            let data = resp.bytes().await?;
            g.put(key, data.clone());
            data
        }
    };

    Ok(data)
}

// Debugging helper function
fn print_test_url(url: &str) {
    use std::borrow::Borrow;
    let spec1 = Spec::new_resize(500, 800, resize::SampleFilter::CatmullRom);
    let spec2 = Spec::new_watermark(20, 20);
    let spec3 = Spec::new_filter(filter::Filter::Marine);
    let image_spec = ImageSpec::new(vec![spec1, spec2, spec3]);
    let s: String = image_spec.borrow().into();
    let test_image = percent_encode(url.as_bytes(), NON_ALPHANUMERIC).to_string();
    println!("Test URL: http://localhost:3000/image/{}/{}", s, test_image);
}
```

This code looks lengthy, but it mainly adds the `retrieve_image` function. For network requests to fetch images, we hash the URL, check the LRU cache, and use `reqwest` to send the request if not found in the cache.

You can run the current code with `cargo run`.

```sh
➜  thumbor git:(main) ✗ RUST_LOG=info cargo run --quiet     

test url: http://localhost:3000/image/CgoKCAj0AxCgBiADCgY6BAgUEBQKBDICCAM/https%3A%2F%2Fimages%2Epexels%2Ecom%2Fphotos%2F1562477%2Fpexels%2Dphoto%2D1562477%2Ejpeg%3Fauto%3Dcompress%26cs%3Dtinysrgb%26dpr%3D3%26h%3D750%26w%3D1260
Sep 08 15:47:40.752  INFO thumbor: Listening on 127.0.0.1:3000
```

To facilitate testing, I added a helper function to generate a test URL. Opening this URL in a browser should return an image identical to the source image, indicating that the network handling part is done.

## Image Processing

Next, we can process the image. Rust has a good low-level `image` library, with many higher-level libraries built around it, including `photon_rs`, which we will use today.

After a quick review of its source code, I don't think it's particularly well-designed, with too many unnecessary memory copies internally. However, its benchmarks show it significantly outperforms PIL/ImageMagick, which is a testament to Rust's powerful performance.

![photon-bench](images/rust-05-02.webp)

Because `photo_rs` is easy to use, we don't need to worry too much about higher performance for now. However, as ambitious developers, we know that one day we might want to replace it with a different image engine, so we design an `Engine` trait:

```rust
// Engine trait: We can add more engines in the future, and only need to replace the engine in the main process
pub trait Engine {
    // Apply a series of ordered processing steps to the engine according to specs
    fn apply(&mut self, specs: &[Spec]);
    // Generate the target image from the engine, note that we use self here, not a reference to self
    fn generate(self, format: ImageOutputFormat) -> Vec<u8>;
}
```

It provides two methods: `apply`, **which applies a series of ordered processing steps to the engine according to specs, and `generate`, which generates the target image from the engine**.

So how do we implement the `apply` method? We can design another trait, so that we can generate corresponding processing for each `Spec`:

```rust
// SpecTransform: If we add more specs in the future, we only need to implement this trait
pub trait SpecTransform<T> {
    // Apply the transform to the image using the op
    fn transform(&mut self, op: T);
}
```

With this idea, we create the `src/engine` directory and add `src/engine/mod.rs`. In this file, we add the trait definitions:

```rust
use crate::pb::Spec;
use image::ImageOutputFormat;

mod photon;
pub use photon::Photon;

// Engine trait: We can add more engines in the future, and only need to replace the engine in the main process
pub trait Engine {
    // Apply a series of ordered processing steps to the engine according to specs
    fn apply(&mut self, specs: &[Spec]);
    // Generate the target image from the engine, note that we use self here, not a reference to self
    fn generate(self, format: ImageOutputFormat) -> Vec<u8>;
}

// SpecTransform: If we add more specs in the future, we only need to implement this trait
pub trait SpecTransform<T> {
    // Apply the transform to the image using the op
    fn transform(&mut self, op: T);
}
```

Next, we create a file `src/engine/photon.rs` to implement the `Engine` trait for `photon`. This file mainly contains some implementation details, which I'll omit here, but you can see the comments:

```rust
use super::{Engine, SpecTransform};
use crate::pb::*;
use anyhow::Result;
use bytes::Bytes;
use image::{DynamicImage, ImageBuffer, ImageOutputFormat};
use lazy_static::lazy_static;
use photon_rs::{
    effects, filters, multiple, native::open_image_from_bytes, transform, PhotonImage,
};
use std::convert::TryFrom;

lazy_static! {
    // Preload the watermark file as a static variable
    static ref WATERMARK: PhotonImage = {
        // You need to copy the corresponding image from my GitHub project to your root directory
        // The include_bytes! macro will directly read the file into the compiled binary at compile time
        let data = include_bytes!("../../rust-logo.png");
        let watermark = open_image_from_bytes(data).unwrap();
        transform::resize(&watermark, 64, 64, transform::SamplingFilter::Nearest)
    };
}

// We currently support the Photon engine
pub struct Photon(PhotonImage);

// Convert from Bytes to Photon structure
impl TryFrom<Bytes> for Photon {
    type Error = anyhow::Error;

    fn try_from(data: Bytes) -> Result<Self, Self::Error> {
        Ok(Self(open_image_from_bytes(&data)?))
    }
}

impl Engine for Photon {
    fn apply(&mut self, specs: &[Spec]) {
        for spec in specs.iter() {
            match spec.data {
                Some(spec::Data::Crop(ref v)) => self.transform(v),
                Some(spec::Data::Contrast(ref v)) => self.transform(v),
                Some(spec::Data::Filter(ref v)) => self.transform(v),
                Some(spec::Data::Fliph(ref v)) => self.transform(v),
                Some(spec::Data::Flipv(ref v)) => self.transform(v),
                Some(spec::Data::Resize(ref v)) => self.transform(v),
                Some(spec::Data::Watermark(ref v)) => self.transform(v),
                // For specs we don't recognize, we do nothing
                _ => {}
            }
        }
    }

    fn generate(self, format: ImageOutputFormat) -> Vec<u8> {
        image_to_buf(self.0, format)
    }
}

impl SpecTransform<&Crop> for Photon {
    fn transform(&mut self, op: &Crop) {
        let img = transform::crop(&mut self.0, op.x1, op.y1, op.x2, op.y2);
        self.0 = img;
    }
}

impl SpecTransform<&Contrast> for Photon {
    fn transform(&mut self, op: &Contrast) {
        effects::adjust_contrast(&mut self.0, op.contrast);
    }
}

impl SpecTransform<&Flipv> for Photon {
    fn transform(&mut self, _op: &Flipv) {
        transform::flipv(&mut self.0)
    }
}

impl SpecTransform<&Fliph> for Photon {
    fn transform(&mut self, _op: &Fliph) {
        transform::fliph(&mut self.0)
    }
}

impl SpecTransform<&Filter> for Photon {
    fn transform(&mut self, op: &Filter) {
        match filter::Filter::from_i32(op.filter) {
            Some(filter::Filter::Unspecified) => {}
            Some(f) => filters::filter(&mut self.0, f.to_str().unwrap()),
            _ => {}
        }
    }
}

impl SpecTransform<&Resize> for Photon {
    fn transform(&mut self, op: &Resize) {
        let img = match resize::ResizeType::from_i32(op.rtype).unwrap() {
            resize::ResizeType::Normal => transform::resize(
                &mut self.0,
                op.width,
                op.height,
                resize::SampleFilter::from_i32(op.filter).unwrap().into(),
            ),
            resize::ResizeType::SeamCarve => {
                transform::seam_carve(&mut self.0, op.width, op.height)
            }
        };
        self.0 = img;
    }
}

impl SpecTransform<&Watermark> for Photon {
    fn transform(&mut self, op: &Watermark) {
        multiple::watermark(&mut self.0, &WATERMARK, op.x, op.y);
    }
}

// photon library doesn't provide a method to convert the image format in memory, so we implement it manually
fn image_to_buf(img: PhotonImage, format: ImageOutputFormat) -> Vec<u8> {
    let raw_pixels = img.get_raw_pixels();
    let width = img.get_width();
    let height = img.get_height();

    let img_buffer = ImageBuffer::from_vec(width, height, raw_pixels).unwrap();
    let dynimage = DynamicImage::ImageRgba8(img_buffer);

    let mut buffer = Vec::with_capacity(32768);
    dynimage.write_to(&mut buffer, format).unwrap();
    buffer
}
```

Alright, the image processing engine is done. We used a watermark image, which you can download from the [GitHub repo](https://github.com/GarryChen-site/medium-repo/blob/main/Rust/thumbor/rust-logo.png) and place in the project root directory. We also add the engine module to `main.rs` and import `Photon`:

```rust
mod engine;
use engine::{Engine, Photon};
use image::ImageOutputFormat;
```

Remember the `TODO` in the code in `src/main.rs`?

```rust
// TODO: Process the image

let mut headers = HeaderMap::new();

headers.insert("content-type", HeaderValue::from_static("image/jpeg"));
Ok((headers, data.to_vec()))
```

We replace this part with the Photon engine we just wrote:

```rust
// Process the image using the image engine
let mut engine: Photon = data
    .try_into()
    .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
engine.apply(&spec.specs);

let image = engine.generate(ImageOutputFormat::Jpeg(85));

info!("Finished processing: image size {}", image.len());
let mut headers = HeaderMap::new();

headers.insert("content-type", HeaderValue::from_static("image/jpeg"));
Ok((headers, image))
```

With this, the entire server process is complete. You can access the full code on the [GitHub repo](https://github.com/GarryChen-site/medium-repo/tree/main/Rust/thumbor). 

I tested the effect with a randomly found image online. Compile the `thumbor` project with `cargo build --release`, then open the logs and run it:

Open the test link in your browser, and you will see the processed image with the watermark and the Marine filter applied in the lower-left corner.

![watermark image](images/rust-05-03.webp)



It worked! This is our Thumbor service, which resizes the image to 500x800, adds a watermark, and applies the Marine filter according to the user's request.

From the logs, you can see that the first request took 400ms because it needed to request the source image, while subsequent requests for the same image hit the cache and took about 200ms.

```sh
Sep 10 10:15:23.415  INFO thumbor: Listening on 127.0.0.1:3000
Sep 10 10:15:32.927  INFO retrieve_image{url="https://images.pexels.com/photos/1562477/pexels-photo-1562477.jpeg?auto=compress&cs=tinysrgb&dpr=3&h=750&w=1260"}: thumbor: Retrieve url
Sep 10 10:15:35.184  INFO thumbor: Finished processing: image size 52590
Sep 10 10:16:06.757  INFO retrieve_image{url="https://images.pexels.com/photos/1562477/pexels-photo-1562477.jpeg?auto=compress&cs=tinysrgb&dpr=3&h=750&w=1260"}: thumbor: Match cache 13782279907884137652
Sep 10 10:16:06.850  INFO thumbor: Finished processing: image size 52590
```


This version is currently not thoroughly optimized, but its performance is already quite good. Moreover, for an image service like Thumbor, there is a CDN (Content Distribution Network) in front of it to handle the load. The server is only accessed when the CDN needs to fetch the original image, so extensive optimization isn't necessary.

Let's review how well we've met our goals. If we exclude the code generated by protobuf, we've written 324 lines of code for the Thumbor project so far:

```sh

➜  thumbor git:(main) ✗ tokei src/main.rs src/engine/* src/pb/mod.rs 
===============================================================================
 Language            Files        Lines         Code     Comments       Blanks
===============================================================================
 Rust                    4          388          311            7           70
===============================================================================
 Total                   4          388          311            7           70
===============================================================================

```

In just over 300 lines of code, we've managed to implement the core part of an image server. Not only that, but we've also fully considered the scalability of the architecture. We've used traits to implement the main image processing flow and introduced caching to avoid unnecessary network requests. Although the code is 50% more than our initial estimate of 200 lines, I believe this further demonstrates Rust's powerful expressive capability.

Furthermore, **by reasonably using protobuf to define interfaces and traits for the image engine, adding new features in the future will be very simple**. We can stack new features like building blocks without affecting the existing functionality, perfectly aligning with the [Open-Closed Principle](https://en.wikipedia.org/wiki/Open%E2%80%93closed_principle).

As a system-level language, Rust uses a unique memory management scheme to manage memory at zero cost. As a high-level language, Rust provides a powerful type system and a comprehensive standard library, helping us write low-coupling, high-cohesion code with ease.

## Summary

Today's discussion on Thumbor is an order of magnitude more challenging than the previous one on HTTPie (the complete code is in the [GitHub repo](https://github.com/GarryChen-site/medium-repo/tree/main/Rust/thumbor)). It's okay if you don't understand every detail, but I believe you'll be further impressed by Rust's expressive power, abstraction capability, and practical problem-solving abilities.

For example, we've separated the specific image processing engine from the main process using the Engine trait, making the main process clean and straightforward. When dealing with protobuf-generated data structures, we extensively used the From/TryFrom trait for data type conversion, which is also a way to achieve decoupling (separation of concerns).

Listening to me speak so smoothly, you might think I didn't make any mistakes while writing this. Actually, I did. When writing the source image retrieval process using Axum, I was beaten by the compiler due to a mistake in using Mutex, which took some time to resolve.

But this kind of beating is very satisfying and enjoyable because I know that such concurrency issues, if leaked into the production environment, would be incredibly difficult to diagnose and fix. The cost would be far greater than the ten minutes of battling with the compiler.

So once you get the hang of it, writing Rust code is absolutely enjoyable. Most errors are caught at compile time, and once your code compiles successfully, you generally don't have to worry about its runtime correctness.

It's precisely because of this that compiling during the early stages of learning Rust is difficult, making the language seem hard to learn. But in reality, it's quite approachable. This might sound contradictory, but it matches my experience: it's challenging to learn because, like a Latin language speaker learning Chinese, **you have to break many of your existing perceptions and embrace new ideas and concepts**. However, with practice and reflection, understanding it becomes second nature.

---