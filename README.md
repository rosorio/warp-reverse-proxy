# warp-reverse-proxy

[![MIT licensed](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
[![GHA Build Status](https://github.com/danielsanchezq/warp-reverse-proxy/workflows/CI/badge.svg)](https://github.com/danielsanchezq/warp-reverse-proxy/actions?query=workflow%3ACI)
[![Docs Badge](https://docs.rs/warp-reverse-proxy/badge.svg)](https://docs.rs/warp-reverse-proxy)

Fully composable [warp](https://github.com/seanmonstar/warp) filter that can be used as a reverse proxy. It forwards the request to the 
desired address and replies back the remote address response.

### Add the library dependency
```toml
[dependencies]
warp = "0.3"
warp-reverse-proxy = "1"
```

### Use it as simple as:
```rust
use warp::{hyper::Body, Filter, Rejection, Reply, http::Response};
use warp_reverse_proxy::reverse_proxy_filter;

async fn log_response(response: Response<Body>) -> Result<impl Reply, Rejection> {
    println!("{:?}", response);
    Ok(response)
}

#[tokio::main]
async fn main() {
    let hello = warp::path!("hello" / String).map(|name| format!("Hello, {}!", name));
    // // spawn base server
    tokio::spawn(warp::serve(hello).run(([0, 0, 0, 0], 8080)));
    // Forward request to localhost in other port
    let app = warp::path!("hello" / ..).and(
        reverse_proxy_filter("".to_string(), "http://127.0.0.1:8080/".to_string())
            .and_then(log_response),
    );
    // spawn proxy server
    warp::serve(app).run(([0, 0, 0, 0], 3030)).await;
}
```


### For more control. You can compose inner library filters to help you compose your own reverse proxy:

```rust
#[tokio::main]
async fn main() {
    let hello = warp::path!("hello" / String).map(|name| format!("Hello port, {}!", name));

    // // spawn base server
    tokio::spawn(warp::serve(hello).run(([0, 0, 0, 0], 8080)));

    let request_filter = extract_request_data_filter();
    let app = warp::path!("hello" / String)
        // build proxy address and base path data from current filter
        .map(|port| (format!("http://127.0.0.1:{}/", port), "".to_string()))
        .untuple_one()
        // build the request with data from previous filters
        .and(request_filter)
        .and_then(proxy_to_and_forward_response)
        .and_then(log_response);

    // spawn proxy server
    warp::serve(app).run(([0, 0, 0, 0], 3030)).await;
}
```

### Requests client initialization

By default, a simple `reqwests::Client` is initialized and used.
In case some specific client configuration need to be used it can be overridden:

```rust
use warp_reverse_proxy::{reverse_proxy_filter, CLIENT as PROXY_CLIENT};

#[tokio::main]
async fn main() {
    let client = reqwest::Client::builder()
        .default_headers(headers)
        .build().expect("client goes boom...");
    PROXY_CLIENT.set(client).expect("client couldn't be set");
    ...
}
```

### Change query header and extra parameters

```rust
use warp::{Filter, hyper::Body, Reply, Rejection, hyper::Response};
use warp_reverse_proxy::{extract_request_data_filter, proxy_to_and_forward_response, Headers};

#[tokio::main]
async fn main() {
    let request_filter = extract_request_data_filter();
    let app = warp::any()
        // build the request with data from previous filters
        .and(request_filter)
        .and_then(|path: warp::filters::path::FullPath, query, method, mut headers: Headers, mut body: warp::hyper::body::Bytes|{
            headers
                // insert header values
                .insert("FOO_HEADER", "Foo content".parse().unwrap())
                .unwrap();
            proxy_to_and_forward_response(
                "http://127.0.0.1:3000".to_string(),
                "".to_string(),
                path,
                query,
                method,
                headers,
                body,
            )
        });

    // spawn proxy server
    warp::serve(app).run(([0, 0, 0, 0], 3030)).await;
}
```
