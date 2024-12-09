---
Title: ASP .NET 5, Npgsql, Linux & Mono 4.2
Slug: asp-net-5-npgsql-linux-mono-4-2
---

# ASP .NET 5, Npgsql, Linux & Mono 4.2

### Background

[Logic Soft](http://www.logicsoft.co.in)'s primary application is a Windows
based application written in C# using Windows.Forms. So when Microsoft
[announced](http://blogs.msdn.com/b/webdev/archive/2015/11/18/announcing-asp-net-5-release-candidate-1.aspx)
ASP .NET 5 RC1, needless to say, we were quite excited.

We picked it up and started our experimentation - writing smaller applications
and porting some of our side projects to the CoreCLR platform. As long as we
didn't go to the place that needed WIN32 APIs, it worked seamlessly.

With a small sum of experience, we took up the task of writing one of our
internal dashboards with it. The aim was to be **completely** cross platform
from the start since the it was to be hosted on our Linux server. The Dashboard
is based on the completely revamped ASP MVC 6 with an Angular JS frontend. It
uses [Npgsql](http://npgsql.org/) to connect to a PostgreSQL database at the
backend.

---

### The tl;dr version

- Npgsql can't work on Linux with CoreCLR RC1 because of issues -
  [#874](https://github.com/npgsql/npgsql/issues/874),
  [#4631](https://github.com/dotnet/corefx/issues/4631),
  [#4652](https://github.com/dotnet/corefx/pull/4652)
- Mono `4.2.1` is **buggy**. Pin version to `4.0.5` instead.
- Pinning version to `4.0.5` on Ubuntu via [suggested
  method](http://www.mono-project.com/docs/getting-started/install/linux/#accessing-older-releases)
  results in `Conflicting distribution` issues when doing `apt-get update`. Pin
  it to `4.0.5.1` instead.

---

### Npgsql on CoreCLR RC1

Because of how easy Visual Studio makes our work, we decided to write the app
on windows but based on CoreCLR. After a few days of adding a major chunk of
features, we decided to test the app out on Linux and instantly it failed with
a `TimeoutException` when trying to open a connection to the Postgres DB. 

```
System.TimeoutException: The operation has timed out.
at Npgsql.NpgsqlConnector.Connect(NpgsqlTimeout timeout)
at Npgsql.NpgsqlConnector.RawOpen(NpgsqlTimeout timeout)
at Npgsql.NpgsqlConnector.Open(NpgsqlTimeout timeout)
at Npgsql.NpgsqlConnector.Open()
at Npgsql.NpgsqlConnectorPool.GetPooledConnector(NpgsqlConnection Connection)
at Npgsql.NpgsqlConnectorPool.RequestConnector(NpgsqlConnection connection)
at Npgsql.NpgsqlConnection.OpenInternal(NpgsqlTimeout timeout)
at Npgsql.NpgsqlConnection.Open()
```

This didn't make sense to me because when I used `psql`, I was able to connect
to the Postgres DB. So I begun investigating.

My investigation led me to [this
issue](https://github.com/npgsql/npgsql/issues/874) filed on Npgsql's repo. It
talks about a problem with the `Socket.Select` method under Linux. The reporter
of the issue opened an associated
[issue](https://github.com/dotnet/corefx/issues/4631) on the CoreFx repo which
was very quickly [fixed](https://github.com/dotnet/corefx/pull/4652) and merged
in. The only nitpick of mine with this was that the fix would come through only
in RC2.  From the latest [standup](https://www.youtube.com/watch?v=UvkJJGVq-b0)
it was made clear that RC2 would not be released till Feb 2016. We wanted to
get this dashboard into production by end of next week. This meant that I had
to look for alternatives.

### Enter Mono (4.2.1)

The alternative here is quite simple - [Mono](http://www.mono-project.com/).
From their site: 

> Sponsored by Xamarin, Mono is an open source implementation of Microsoft's
> .NET Framework based on the ECMA standards for C# and the Common Language
> Runtime. A growing family of solutions and an active and enthusiastic
> contributing community is helping position Mono to become the leading choice
> for development of cross platform applications.

The first thing I did was to install the `mono-complete` package as advised on
their very clear [installation
page](http://www.mono-project.com/docs/getting-started/install/linux/#debian-ubuntu-and-derivatives).
This installed Mono 4.2.1. Along with this I installed the associated `dnx`
too.

Once this was done, I used [yeoman](http://yeoman.io/) and the [omnisharp
generators](https://www.npmjs.com/package/generator-aspnet) for ASP .NET 5 to
generate a simple console application. I added Npgsql as a dependency and wrote
just enough code to open a connection to the Postgres DB and when I ran it, I
could connect to the DB with no `TimeoutException` biting my behind. I was
happy.  With this, my expectation was that I could just run a `dnu restore` on
my project, and get cracking.

After running a `dnu restore`, I quickly ran `dnx web` to see the ever friendly
`Now listening on: http://localhost:5000` message greet me. With a great sum of
enthusiasm, I visited `http://localhost:5000` on my browser only to see it
never finish. I switched back to the console to check and I saw that the
request processing was stuck up at one stage:

```
> dnx web
Hosting environment: Production
Now listening on: http://*:5004
Application started. Press Ctrl+C to shut down.
info: Microsoft.AspNet.Hosting.Internal.HostingEngine[1]
      Request starting HTTP/1.1 GET http://localhost:5004/  
info: Microsoft.AspNet.Mvc.Controllers.ControllerActionInvoker[1]
      Executing action method MvcSample.Web.HomeController.Index with arguments () - ModelState is Valid'
```
No matter how long I'd let it be, the response wouldn't come through.
Roadblock.

After a while of fiddling with `DNX` by setting `DNX_TRACE=1`, I decided that
the best way of understanding better would be to run a "Microsoft certified"
basic application. These can be found for over on the [home
repo](https://github.com/aspnet/home) under the aspnet organization.  Here, in
the **samples** directory one can find applications for each of the releases
Microsoft has made. All the way from `-beta4` to `-RC1-update1` to the cutting
edge one as well. This would be a good place for me to start debugging the
issue I was facing, I thought.

Once I cloned the repo, I went into the `1.0.0-rc1-update1/HelloMvc` folder and
did a `dnu restore` to bring down dependencies. Post this, I ran `dnx web` and
navigated to the URL. To my surprise I **again** hit the same problem. It was
getting stuck at precisely the same place as earlier:

```
> dnx web
Hosting environment: Production
Now listening on: http://*:5004
Application started. Press Ctrl+C to shut down.
info: Microsoft.AspNet.Hosting.Internal.HostingEngine[1]
      Request starting HTTP/1.1 GET http://localhost:5004/  
info: Microsoft.AspNet.Mvc.Controllers.ControllerActionInvoker[1]
      Executing action method MvcSample.Web.HomeController.Index with arguments () - ModelState is Valid'
```

Initially, I was quite skeptical of myself and was convinced that I was doing
something wrong. So I double checked everything again - `dnx`, `dnu` and `dnvm`
versions. I reinstalled everything and tried again but alas - Same issue.

With nothing else possible, I opened an issue on the aspnet/home repo:
[#1161](https://github.com/aspnet/Home/issues/1161) detailing all the steps
that I followed and the problem that I was facing. Following this, I jumped on
to the [JabbR chat room](https://jabbr.net/#/rooms/aspnetvnext) to see if I
could find anyone. I posted about the issue there as well and one of the core
guys at Microsoft - [David Fowler](https://github.com/davidfowl) asked me to
try out Mono 4.0.5 instead of 4.2.1. He went on to mention that 4.2.1 was
filled with quite a few bugs and the recommendation is to pin the version to
4.0.5 until the bugs in 4.2.1 were fixed.

### Exit Mono (4.2.1)

Now, on my Ubuntu box, I had 4.2.1 installed which I now had to uninstall in
order to replace it with 4.0.5. As with uninstalling any other package, I
followed the usual steps: 

```
sudo apt-get remove mono-complete
sudo apt-get purge mono-complete
sudo apt-get autoremove
```

### Enter Mono (4.0.5?)

The Mono install page has a section on [how to access older
releases](http://www.mono-project.com/docs/getting-started/install/linux/#accessing-older-releases)
which was the one I followed in order to pin my Mono version to 4.0.5. I edited
the `/etc/apt/sources.list.d/mono-xamarin.d` and changed the line to 

```shell
deb http://download.mono-project.com/repo/debian wheezy/snapshots/4.0.5 main
```

Following this I did a `sudo apt-get update` to bring update the package
listing and soon enough ended up with this: 

```bash
W: Conflicting distribution: http://download.mono-project.com wheezy/snapshots/4.0.5 InRelease (expected wheezy/snapshots but got wheezy)
```

After some (rather hard) googling, I came across [this
issue](https://bugzilla.xamarin.com/show_bug.cgi?id=24902) filed on Xamarin's
bugzilla. However this was for installing the `3.12.0` version and a workaround
was suggested to suffix a `/.` to the end of the version. I then updated the `mono-xamarin.d` file
to have 

```
deb http://download.mono-project.com/repo/debian wheezy/snapshots/4.0.5/. main
```

instead but to no avail. I ended up with the same error message again.

After some digging around on Mono's [download
server](http://download.mono-project.com/repo/debian) for
[4.0.5](http://download.mono-project.com/repo/debian/dists/wheezy/snapshots/4.0.5/),
I noticed that the [`InRelease` file](http://download.mono-project.com/repo/debian/dists/wheezy/snapshots/4.0.5/InRelease)
for 4.0.5 had the Suite listed as 4.0.5.1. Off a whim, I tried replacing
4.0.5 with 4.0.5.1 in my `mono-xamarin.d` file. 

```
deb http://download.mono-project.com/repo/debian wheezy/snapshots/4.0.5.1/. main
```
Running `sudo apt-get update` post this didn't leave me with the dreaded
"`Conflicting distribution`" error anymore.  After this I just installed the
`mono-complete` package as before.

---

**Note:** The first time I did this, somehow `mono-runtime` didn't get removed
and I was having package clash problems when trying to install the
`mono-complete` package after pinning 4.0.5.1 (since the `mono-runtime` for 4.2.1
was still installed). I had to explicitly remove the `mono-runtime` package for
it to work.

---

With mono 4.0.5 installed, I got back to my project and booted it up again
with `dnx web`. I visited the URL and finally

```
> dnx web
Hosting environment: DEVELOPMENT
Now listening on: http://localhost:5000
Application started. Press Ctrl+C to shut down.
info: Microsoft.AspNet.Hosting.Internal.HostingEngine[1]
      Request starting HTTP/1.1 GET http://localhost:5000/  
info: Microsoft.AspNet.Mvc.Controllers.ControllerActionInvoker[1]
      Executing action method SessyDash.Controllers.HomeController.Index with arguments () - ModelState is Valid'
info: Microsoft.AspNet.Mvc.ViewFeatures.ViewResultExecutor[1]
      Executing ViewResult, running view at path /Views/Home/Index.cshtml.
info: Microsoft.Extensions.DependencyInjection.DataProtectionServices[0]
      User profile is available. Using '/home/shrayasr/.local/share/ASP.NET/DataProtection-Keys' as key repository; keys will not be encrypted at rest.
info: Microsoft.AspNet.Mvc.Infrastructure.MvcRouteHandler[2]
      Executed action SessyDash.Controllers.HomeController.Index in 0.4223ms
info: Microsoft.AspNet.Hosting.Internal.HostingEngine[2]
      Request finished in 0.468ms 200 text/html; charset=utf-8
```

The page rendered successfully and all the connections to the PostgreSQL DB
went through as well. Success!

### Conclusion

This meant that the set of features that we had written for the dashboard now
worked on both Windows (via CoreCLR) and on Linux (via Mono). This is really
exciting for us as a team since we've only worked on the Windows platform for
so long. Being able to translate those skills (almost) directly to other
platforms _with_ Microsoft's support is a very **very** big win.

ASP .NET 5 is a really interesting platform to work on. It feels like Microsoft
is taking a wonderful way forward. Personally I've always enjoyed programming
with C# and this just makes that experience that much better.

However, this being still a nacent platform, has a few rough edges and adopting
into it this early has its own set of problems related to documentation and
finding workarounds. But the dynamic team at Microsoft really eases that out
too. The [JabbR room](https://jabbr.net/#/rooms/AspNetvNext) seems like a very
interesting place to be on and I intend to be a frequent visitor there helping
people and seeking help too in the future.

Heres to more adventures with CoreCLR, ASP .NET 5 et al. Cheers!
