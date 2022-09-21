# fly-replay-header-caddy
A sample Caddy reverse proxy setup that leverages the Fly Replay Header to route requests globally

## Using Caddy as a reverse proxy on Fly.io

... is great, if you keep a couple gotchas in mind.

### Caddy v1 SEO

When looking up literally anything about Caddy, you'll want to rely nearly exclusively on the (excellent) [docs](https://caddyserver.com/docs/) or very recent content because otherwise you're likely to find information about Caddy v1 instead of the v2 you should be using.

### Double HTTPS

One of the greatest features of Caddy as a turnkey web server and reverse proxy is its ability to [use HTTPS automatically and by default](https://caddyserver.com/docs/automatic-https#automatic-https). It's an absolute game-changer. Except on Fly.io which already does that for you. You'll want to disable this feature in Caddy with `auto_https off`.

### Sketchy health checks

Because they determine whether to continue or roll back new versions of an app, you almost certainly want to configure Fly.io's health checks to test whether Caddy itself works, rather than have it pass the check "upstream" to another app. This can be accomplished in several ways, such as serving a permanent `2011` response for a specific route (e.g. `respond /health-check "OK" 200` in Caddy) and then configuring HTTP health checks in Fly.io for it:

```toml
[[services.http_checks]]
    grace_period = "1s"
    interval = "15s"
    method = "get"
    path = "/health-check"
    protocol = "http"
    restart_limit = 0
    timeout = "2s"
    tls_skip_verify = true
```

### Port-mapping shenanigans

By default, the `fly.toml` config exposes ports 80 and 443 to the internet, and maps them to 8080 internally. This means that Caddy has to be configured to listen on 8080 as a public port instead of its default 80. And because Fly.io terminates TLS for us, everything within its network can be assumed to be regular HTTP, I think. In practice this means we can default Caddy's HTTP port to 8080 with the `http_port 8080` global option, and configure the "site address" to use 8080 as well (just to be sure). Maybe something like this:

```Caddyfile
:8080 {
    reverse_proxy * https://my-app.fly.dev:443
}
```

### Websocket origin violation

If you use Caddy as a reverse proxy for static content, things should just work. If you rely on websockets, then the upstream server might rightfully take offense to the redirection. Here's a sample error with Phoenix, with a polite suggestion as to how to fix it:

```bash
[error] Could not check origin for Phoenix.Socket transport.
Origin of the request: <redacted>
This happens when you are attempting a socket connection to
a different host than the one configured in your config/
files. For example, in development the host is configured
to "localhost" but you may be trying to access it from
"127.0.0.1". To fix this issue, you may either:
  1. update [url: [host: ...]] to your actual host in the
     config file for your current environment (recommended)
  2. pass the :check_origin option when configuring your
     endpoint or when configuring the transport in your
     UserSocket module, explicitly outlining which origins
     are allowed:
        check_origin: ["https://example.com",
                       "//another.com:888", "//other.com"]
```
## References

* [docs](https://caddyserver.com/docs/) and [Caddyfile concepts](https://caddyserver.com/docs/caddyfile/concepts) -- caddyserver.com
* [hosting a static site on fly.io with nix and caddy](https://mat.services/posts/static-site-with-nix-and-caddy/) -- mat.services
* [Run a Static Website](https://fly.io/docs/languages-and-frameworks/static/) -- Fly.io