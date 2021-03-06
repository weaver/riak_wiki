## Demo

The [[demo|https://github.com/basho/webmachine/tree/master/demo]]
application from the Webmachine repository provides two basic example
resources:

* [[webmachine_demo_fs_resource|https://github.com/basho/webmachine/blob/master/demo/src/webmachine_demo_fs_resource.erl]] - an example of a read/write filesystem server showing several interesting features and supporting GET, PUT, DELETE, and POST
* [[webmachine_demo_resource|https://github.com/basho/webmachine/blob/master/demo/src/webmachine_demo_resource.erl]] - an example of a simple resource demonstrating content negotiation, basic auth, and some caching headers.

The rest of this page walks through the code responsible for the
[[webmachine_demo_resource|https://github.com/basho/webmachine/blob/master/demo/src/webmachine_demo_resource.erl]].

## Path Dispatch

Mapping a path to a resource requires updating the dispatch
settings. In the demo application this is controlled in
[[priv/dispatch.conf|https://github.com/basho/webmachine/blob/master/demo/priv/dispatch.conf]]:

```erlang
%%-*- mode: erlang -*-
{["demo", '*'], webmachine_demo_resource, []}.
{["fs", '*'], webmachine_demo_fs_resource, [{root, "/tmp/fs"}]}.
```

With the above dispatch settings the `demo` and `demo_fs` resources
can be accessed from the `/demo` and `/fs` paths respectively.

## HTML Content

The simplest working resource exports only one function in addition to
`init/1`:

```erlang
-module(webmachine_demo_resource).
-export([init/1, to_html/2]).
-include_lib("webmachine/include/webmachine.hrl").

init([]) -> {ok, undefined}.

to_html(ReqData, Context) -> {"Hello, new world", ReqData, Context}.
```

That's really it -- a working Webmachine resource. That resource will
respond to all valid GET requests with the exact same response.

Many interesting bits of HTTP are handled automatically by
Webmachine. For instance, if a client sends a request to that trivial
resource with an `Accept` header that does not allow for a `text/html`
response, they will receive a `406 Not Acceptable`.

## Plaintext Content

Suppose I wanted to serve a plaintext client as well. I could note
that I provide more than just HTML:

```erlang
content_types_provided(ReqData, Context) ->
   {[{"text/html", to_html},{"text/plain",to_text}], ReqData, Context}.
```

I already have my HTML representation produced, so I add a text one:
(and while I'm at it, I'll show that it's trivial to produce dynamic
content as well)

```erlang
to_text(ReqData, Context) ->
    Path = wrq:disp_path(ReqData),
    Body = io_lib:format("Hello ~s from Webmachine.~n", [Path]),
    {Body, ReqData, Context}.
```

## Content Negotiation

Now that this resource provides multiple media types, it automatically
performs content negotiation:

```bash
$ telnet localhost 8000
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
GET /demo/a/resource/path HTTP/1.1
Accept: text/plain

HTTP/1.1 200 OK
Vary: Accept
Server: MochiWeb/1.1 WebMachine/0.97
Date: Sun, 15 Mar 2009 02:54:02 GMT
Content-Type: text/plain
Content-Length: 39

Hello a/resource/path from Webmachine.
```

## Authorization

What about authorization? Webmachine resources default to assuming the
client is authorized, but that can easily be overridden. Here's an
overly simplistic but illustrative example:

```erlang
is_authorized(ReqData, Context) ->
    case wrq:disp_path(ReqData) of
        "authdemo" ->
            case wrq:get_req_header("authorization", ReqData) of
                "Basic "++Base64 ->
                    Str = base64:mime_decode_to_string(Base64),
                    case string:tokens(Str, ":") of
                        ["authdemo", "demo1"] ->
                            {true, ReqData, Context};
                        _ ->
                            {"Basic realm=Webmachine", ReqData, Context}
                    end;
                _ ->
                    {"Basic realm=Webmachine", ReqData, Context}
            end;
        _ -> {true, ReqData, Context}
    end.
```

With that function in the resource, all paths except `/authdemo` from
this resource's root are authorized. For that one path, the UA will be
asked to do basic authorization with the user/pass of
authdemo/demo1. It should go without saying that this isn't quite the
same function that we use in our real apps, but it is nice and simple.

To see authorization in action point your browser at
`http://localhost:8000/demo/authdemo` with the demo app running:

```bash
$ curl -v http://localhost:8000/demo/authdemo
> GET /demo/authdemo HTTP/1.1
> Host: localhost:8000
> Accept: */*
>
< HTTP/1.1 401 Unauthorized
< WWW-Authenticate: Basic realm=Webmachine
< Server: MochiWeb/1.1 WebMachine/0.97
< Date: Sun, 15 Mar 2009 02:57:43 GMT
< Content-Length: 0

$ curl -v -u authdemo:demo1 http://localhost:8000/demo/authdemo
* Server auth using Basic with user 'authdemo'
> GET /demo/authdemo HTTP/1.1
> Authorization: Basic YXV0aGRlbW86ZGVtbzE=
> Host: localhost:8000
> Accept: */*
>
< HTTP/1.1 200 OK
< Vary: Accept
< Server: MochiWeb/1.1 WebMachine/0.97

< Date: Sun, 15 Mar 2009 02:59:02 GMT
< Content-Type: text/html
< Content-Length: 59
<
<html><body>Hello authdemo from Webmachine.
</body></html>
```

## HTTP Caching Support

HTTP caching support is also quite easy, with functions allowing
resources to define (e.g.) `last_modified`, `expires`, and
`generate_etag`. For instance, since representations of this resource
vary only by URI Path, I could use an extremely simple entity tag
unfit for most real applications but sufficient for this example:

```erlang
generate_etag(ReqData, Context) -> {wrq:raw_path(ReqData), ReqData, Context}.
```

Similarly, here's a trivial expires rule:

```erlang
expires(ReqData, Context) -> {{{2021,1,1},{0,0,0}}, ReqData, Context}.
```

And now the response from our earlier request is appropriately tagged:

```bash
HTTP/1.1 200 OK
Vary: Accept
Server: MochiWeb/1.1 WebMachine/0.97
Expires: Fri, 01 Jan 2021 00:00:00 GMT
ETag: /demo/authdemo
Date: Sun, 15 Mar 2009 02:59:02 GMT
Content-Type: text/html
Content-Length: 59

<html><body>Hello authdemo from Webmachine.
</body></html>
```

For more details, read the source of the resources linked at the top
of this page.
