---
Title: Hiccups with Hosting ASP .NET 5 apps on Linux (RC1)
Slug: aspnet5-hosting-hiccups-nginx-kestrel
---

# Hiccups with Hosting ASP .NET 5 apps on Linux (RC1)

### Background

After the [recent]({{< ref "asp-net-5-npgsql-linux-mono-4-2.md" >}}) dance with
multiple Mono versions, the next step was to actually host the application.

Even though the app is largely still unfinished, I wanted to give it a shot on
the staging instance to get a feel for things. 

The general way to host apps on a Linux box is to have a NGINX reverse proxy
in front, listening to different names on port 80 and to proxy the requests to
internally running servers. This is the same approach we followed to host our
Kestrel app behind NGINX. 

The NGINX configuration for forwarding requests to Kestrel was simple enough: 

```conf
server {
  listen 80;
  server_name appname.domainname.com;

  location / {
    proxy_pass http://localhost:5000;
  }
}
```

Now when I hit http://appname.domainname.com from the browser, The request is
forwarded prompty to Kestrel who replied with this:

```text
info: Microsoft.AspNet.Hosting.Internal.HostingEngine[1]
      Request starting HTTP/1.0 GET http://127.0.0.1:5000/  
info: Microsoft.AspNet.Mvc.Controllers.ControllerActionInvoker[1]
      Executing action method SessyDash.Controllers.HomeController.Index with arguments () - ModelState is Valid'
info: Microsoft.AspNet.Mvc.ViewFeatures.ViewResultExecutor[1]
      Executing ViewResult, running view at path /Views/Home/Index.cshtml.
info: Microsoft.AspNet.Mvc.Infrastructure.MvcRouteHandler[2]
      Executed action SessyDash.Controllers.HomeController.Index in 0.0003ms
info: Microsoft.AspNet.Hosting.Internal.HostingEngine[2]
      Request finished in 0.0004ms 200 text/html; charset=utf-8
```

### The Problem

My expectation after seeing this was that the request had finished processing
and expected the browser to have rendered my index page. However that wasn't
the case. The browser seem to have blocked on something. It only returned after
a timeout and that too only with a partial response.

The first thing I did was to check if I was able to fetch a response from the
locally running server. I visited http://localhost:5000 in the browser and
Voilà - The entire app came up.

At the same time, Kestrel had replied with: 

```text
info: Microsoft.AspNet.Hosting.Internal.HostingEngine[1]
      Request starting HTTP/1.1 GET http://localhost:5000/  
info: Microsoft.AspNet.Mvc.Controllers.ControllerActionInvoker[1]
      Executing action method SessyDash.Controllers.HomeController.Index with arguments () - ModelState is Valid'
info: Microsoft.AspNet.Mvc.ViewFeatures.ViewResultExecutor[1]
      Executing ViewResult, running view at path /Views/Home/Index.cshtml.
info: Microsoft.AspNet.Mvc.Infrastructure.MvcRouteHandler[2]
      Executed action SessyDash.Controllers.HomeController.Index in 0.0021ms
info: Microsoft.AspNet.Hosting.Internal.HostingEngine[2]
      Request finished in 0.0021ms 200 text/html; charset=utf-8
```

Notice something? Both the dumps are **identical**. Yet why was my browser
waiting on something in the first case? To understand better, I wanted to see
the response I got when I queried http://appname.domainname.com

```
$ curl http://appname.domainname.com
```

A part of the response came through following which curl never returned. I
looked at Kestrel's console and surprisingly enough Kestrel says that the
request was finished in `x ms`. How could Kestrel be saying that the response 
went through but curl isn't showing me the entire HTML page? Interesting!

After some digging around, I found [this
issue](https://github.com/aspnet/KestrelHttpServer/issues/418) filed on the
KestrelHttpServer repo. The problem was that Kestrel would hang if there was a
`Connection: close` header sent to it.

### Verification

I wanted to test this out so I added a few lines of code to the `Index()`
action in my `HomeController`.

```CSharp
System.Console.WriteLine("HEADERS ===>");
Request.Headers.ToList().ForEach(header => {
    System.Console.WriteLine($"{header.Key} : {header.Value}");
});
System.Console.WriteLine("<=== HEADERS");
```

This assumes that you are `using` the `System.Linq` namespace.

With this, I could begin debugging. First I started by hitting the server
directly, i.e.: 

```
$ curl http://localhost:5000
```

The relevant section of the output was such

```text
HEADERS ===>
Accept : */*
Host : localhost:5000
User-Agent : curl/7.35.0
<=== HEADERS
```

Ok. Moving on. Time to hit it via the associated name.

```
$ curl http://appname.domainname.com
```

and

```text
HEADERS ===>
Connection : close
Accept : */*
Host : appname.domainname.com
User-Agent : curl/7.35.0
<=== HEADERS
```

Ba Bam! Routing the request via NGINX was adding a `Connection: Close` header.
This confirms that I was indeed facing the
[issue](https://github.com/aspnet/KestrelHttpServer/issues/418) filed by Daniel

### Resolution

From here on out, I had 2 ways to solve the problem: 

1. Download the RC1 Kestrel source, patch it for myself and use that in all my
   projects
2. Update the NGINX configuration

Maintaining a fork of the RC1 Kestrel code ([like Daniel
has](https://github.com/aspnet/KestrelHttpServer/issues/418#issuecomment-158822228))
for these purposes could end up becoming unwieldy. I decided to change the
NGINX configuration instead.

The fix is pretty simple. Set the `Connection` header to `keep-alive`. And tell
it to proxy the request using HTTP/1.1. This can be done in NGINX by adding the
following line to the `location` block

```
proxy_set_header connection keep-alive;
proxy_http_version 1.1;
```

So the full configuration block for appname.domainname.com becomes

```conf
server {
  listen 80;
  server_name appname.domainname.com;

  location / {
    proxy_http_version 1.1;
    proxy_set_header connection keep-alive;
    proxy_pass http://localhost:5000;
  }
}
```

After changing the configuration, restart NGINX.

```
$ sudo service nginx restart
```

and then, hit the server via the name

```
$ curl http://appname.domainname.com
```

This time, on the Kestrel logs we see 

```text
HEADERS ===>
Connection : Keep-Alive
Accept : */*
Host : appname.domainname.com
User-Agent : curl/7.35.0
<=== HEADERS
```

Also we see that the request has completed, yay! :)

### Understanding more - A deep dive into the problem

Now that the **requirement** of hosting the application on kestrel behind NGINX
is done, let us delve a little bit into the problem.

David mentions that the fix went in `e4fd91b`
([link](https://github.com/aspnet/KestrelHttpServer/commit/e4fd91bb68f535801ca8a79aa453ea3fb3f448fe)).
Looking at this, I could understand that the file - `Frame.cs` is the one
responsible for it. Inspecting `Frame.cs`, we deduce that the function in
question is `RequestProcessingAsync`. The comment for which clearly states its
purpose

> **Primary loop which consumes socket input, parses it for protocol framing,
> and invokes the application delegate** for as long as the socket is intended
> to remain open.  The resulting Task from this loop is preserved in a field
> which is used when the server needs to drain and close all currently active
> connections.

Now let us look at the RC1 codebase. The easiest way to browse this codebase
would be from the
[releases](https://github.com/aspnet/KestrelHttpServer/releases/tag/1.0.0-rc1)
page. From here we find that the commit that was tagged RC1 was
`101f2a` ([link](https://github.com/aspnet/KestrelHttpServer/commit/101f2a71809fd92176835f0d7925c924b7121948)).
From the commit, it is easy for us to [browse the
codebase](https://github.com/aspnet/KestrelHttpServer/tree/101f2a71809fd92176835f0d7925c924b7121948)
at that commit.

Lets pick up
`Frame.cs` ([link](https://github.com/aspnet/KestrelHttpServer/blob/101f2a71809fd92176835f0d7925c924b7121948/src/Microsoft.AspNet.Server.Kestrel/Http/Frame.cs))
from the repo and look at the
`RequestProcessingAsync` ([link](https://github.com/aspnet/KestrelHttpServer/blob/101f2a71809fd92176835f0d7925c924b7121948/src/Microsoft.AspNet.Server.Kestrel/Http/Frame.cs#L145))
function - specifically, [lines 201 -
204](https://github.com/aspnet/KestrelHttpServer/blob/101f2a71809fd92176835f0d7925c924b7121948/src/Microsoft.AspNet.Server.Kestrel/Http/Frame.cs#L201-L204).
We see here that there is a `while` statement awaiting on the request body to
be read. 

```csharp
while (await RequestBody.ReadAsync(_nullBuffer, 0, _nullBuffer.Length) != 0)
{
  // Finish reading the request body in case the app did not.
}
```

Keeping that in mind, let us recall our initial NGINX configuration. This is
how it looks:

```conf
server {
  listen 80;
  server_name appname.domainname.com;

  location / {
    proxy_pass http://localhost:5000;
  }
}
```

We see that we were using `proxy_pass`.
[This](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass)
proxies all the requests matching the given location to the URL specified.
Looking at the remaining possible `proxy_*` commands, we land upon something
interesting - `proxy_http_version`
([link](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_http_version)).
From the docs, I quote: 

> Sets the HTTP protocol version for proxying. **By default, version 1.0 is
> used. Version 1.1 is recommended for use with keepalive connections** and NTLM
> authentication.

Now we understand that the request sent from NGINX to Kestrel would be a
**HTTP/1.0** request. With this in mind, let us understand some of the key
differences between HTTP/1.1 and HTTP/1.0. 

The section on persistent connections in [this
document](http://www8.org/w8-papers/5c-protocols/key/key.html#SECTION00052000000000000000)
speaks about the difference we are interested in. I quote:

> HTTP/1.0, in its documented form, made no provision for persistent
> connections. 

We can even pick up the [specification for
HTTP/1.0](http://www.w3.org/Protocols/HTTP/1.0/spec.html) and we wouldn't find
anything on persistent connections. They were a concept that was introduced
in HTTP/1.1. The [spec for
HTTP/1.1](http://www.w3.org/Protocols/rfc2616/rfc2616-sec8.html) confirms this.

For clarity sake, let us recollect the facts till now: 

- We have code in `Frame.cs` (RC1) that awaits on the remaining body to be read
  if the app didn't
- NGINX proxies requests as per the HTTP/1.0 standard
- The HTTP/1.0 standard has no idea of persistent connections. It expects a
  connection to be created before every request and the server to close the
  connection after sending the response

Putting all of these things together, we get a clearer understanding of the
problem. When we hit `http://appname.domainname.com`, NGINX proxies the request
to Kestrel with a `Connection: close` header (Because of HTTP/1.0). Part of the
response is sent back by Kestrel and then it hits the `await` to wait to read
the remaining part of the response. But it never resumes from there on since
the connection is already closed by the server and this resulted in the partial
response.

The
[fix](https://github.com/aspnet/KestrelHttpServer/issues/418#issuecomment-158822228),
once the problem is identified is really trivial. Wrap that entire `while` loop
within an `if (_keepAlive)` statement so that it would only wait if and only if
`Connection: Keep-Alive` was set. The [actual
fix](https://github.com/aspnet/KestrelHttpServer/commit/e4fd91bb68f535801ca8a79aa453ea3fb3f448fe)
that went in also more or less fixes it in the same fashion.

### Conclusion

Hitting this problem means that we need to remember to set the `Connection`
header to `Keep-Alive` along with the setting the HTTP version to `1.1` from
the NGINX side. Once RC2 drops however, we aren't _required_ to do this.
However Keep-Alive is an improvement and it is better to anyway do this.

One of my key learnings from this is that Software is much more complex that it
is made out to be. There will be cases where the code we write will be used
by other code out there. Just using Kestrel in this case didn't have problems
but putting NGINX in front of it, exposed a flaw. This, I felt is a great
representation of how putting different components can result in problems.

When I set out to host my application behind NGINX, I didn't think that I had
to understand how NGINX proxies requests and how the HTTP/1.0 and HTTP/1.1
protocols work. But I ended up learning about them thanks to this 

:)

Once again, major thanks goes out to the many wonderful people working on ASP
.NET 5. The fact that they are doing development in the open allows for
learning opportunities such as these. Cheers, guys.
