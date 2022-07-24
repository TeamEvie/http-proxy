# teamevie-http-proxy

This is our fork of `http-proxy` a ratelimited HTTP proxy in front of the Discord API, making use
of [twilight]. The only difference is that we don't allow the use of multiple applications and only use the default token set to support Discord libraries that forcefully send an authorization header making the proxy think it's another bot with a token of `""`.

## Use

This proxy is not a "real" HTTP `CONNECT` TCP proxy, it instead mocks the
Discord API.

Using it is as easy as changing the base of all routes from `discord.com` to
`localhost:3000`, like so:

```bash
# Previously
$ curl https://discord.com/api/users/@me
# With the proxy
$ curl http://localhost:3000/api/users/@me

# Or other API versions
$ curl http://localhost:3000/users/@me
$ curl http://localhost:3000/api/users/@me
$ curl http://localhost:3000/api/v8/users/@me
```

The proxy actively supports routes from API v9, but will also try to request
the corresponding routes on older or newer API versions if you so request in
the URL.

`twilight_http` natively supports using `twilight_http_proxy`, so you can use
it like this:

```rust
use twilight_http::Client;
use std::error::Error;

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error + Send + Sync>> {
    let client = Client::builder()
        .proxy("localhost:3000", true)
        .ratelimiter(None)
        .build();

    Ok(())
}
```

This will use the running proxy, skip the ratelimiter (since the proxy does
ratelimiting itself), and will request over HTTP.

If you are using a different Discord API client, make sure that you are not
ratelimiting outgoing requests, because the proxy will do this instead. Very
short HTTP client timeouts may also cause issues with longer ratelimits.

### Running via Docker

Prebuilt Docker images are published on [Docker Hub].

```sh
$ docker run -itd -e DISCORD_TOKEN="my token" -p 3000:80 ghcr.io/teamevie/http-proxy:metricsamd64
```

This will set the discord token to `"my token"` and map the bound port to port
3000 on the host machine.

Images come in multiple different variants for metrics and supported
architectures. You can use these with their corresponding image tags found on
the [Docker Hub tags page][docker-hub-tags].

### Running via Cargo

Build the binary:

```sh
$ cargo build --release
$ DISCORD_TOKEN="my token" PORT=3000 ./target/release/twilight_http_proxy
```

This will set the discord token to `"my token"` and bind to port 3000.

### Additional configuration

HTTP2 may cause issues with high concurrency (i.e. many concurrent requests).
If you encounter frequent error logs related to this, force the use of HTTP1 by
setting `DISABLE_HTTP2` to any value when running the proxy.

## Prometheus metrics

The HTTP proxy can expose prometheus metrics when compiled with the
`expose-metrics` feature. These metrics are then available on the `/metrics`
endpoint.
You can set the metrics key used for the histogram data by setting the
`METRIC_KEY` environment variable.

The exported histogram includes timing percentiles, response status codes,
request path and request method. Calls to the metrics endpoint itself are not
included in the metrics.

## Error behaviour

If processing an incoming request fails, the proxy will respond with a 5xx
status code and a helpful error message in the response body. Currently, these
status codes include:

- `500` if the proxy generates an invalid URI or the ratelimiter fails
  internally
- `501` if the client requested an unsupported API path or used an unsupported
  HTTP method
- `502` if the request made by the proxy fails

[twilight]: https://github.com/twilight-rs/twilight
