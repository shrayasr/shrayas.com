---
Title: Generate a quick AWS dashboard using LINQPad
Slug: generate-a-quick-aws-dashboard-using-linqpad
---

# Generate a quick AWS dashboard using LINQPad

## Context

Logic Soft's mainline product -- [WINBDS](http://logicsoft.co.in/product) is an
on premise software i.e. we install it at the client location and we support
them remotely.

Since a while now,this has been seeing a change. For small installations,
usually an instance on the cloud works out cheaper and better. So we've begun
suggesting people to move to AWS when the installation is small enough.

Turns out, if we're diligent about turning the instances off when we don't need
them, a windows box on AWS is actually pretty affordable. It also happens that
SQL Server Express more than suffices for such installations which is free
(with restrictions) 

This coupled with a small tool we've built, to turn on and off instances,
actually has resulted in a win-win situation for both our clients and us.

This now means that we need a way to quickly monitor the status of the
instances that we're hosting. We're not hosting so many instances that I see
the necessity to have a full blown monitoring dashboard nor are we hosting too
less instances for me to check on them individually.

I needed a low tech solution that does the job.

## Enter LINQPad

If you're a C# developer and haven't invested time and money in LINQPad, then
you should stop reading this blog and get right to it. It is one of those tools
that increases productivity by 10 fold once you've gotten attached to it.

For the uninitiated, LINQPad is a program that allows you to quickly play
around with C#/F#/VB expressions/statements/programs -- A scratchpad, if you
may. Think of it like a playground with great tools and features at your
disposal.

Go download it over on [http://www.linqpad.net](http://www.linqpad.net) and play with it.

## Solution

Since I almost always have a LINQPad window running, I decided to write my
dashboard within LINQPad itself. If you're contemplating reading on because you
think that the idea is nonsense,
[think](https://twitter.com/Nick_Craver/status/822268228122669057)
[again](https://twitter.com/Nick_Craver/status/852498557714255873).

Let's start off by installing the
[AWSSDK.EC2](https://www.nuget.org/packages/AWSSDK.EC2/) package from NuGet.
LINQPad has NuGet integration built in so we can install packages for each of
the queries that we write.

We have instances across regions, so let's use a list to keep track of the
regions we're interested in:

```cs
var regions = new List<RegionEndpoint>
{
  RegionEndpoint.USEast1,
  RegionEndpoint.EUWest1,
  RegionEndpoint.APSouthEast1
}
```

For each of these regions, let's create an AWS client

```cs
foreach (var region in regions)
{
  var client = new AmazonEC2Client("<AWS Access Key>", "<AWS Secret Access Key>", region);
```

For the `Access Key` and `Secret Access Key`, the recommended practice is to
create a user specifically for this purpose from the [IAM
console](https://console.aws.amazon.com/iam/home#/home) and use those
credentials.

### Creating users and policies on IAM

IAM is AWS' identity management platform. Here you can create users, attach
policies, manage roles, etc. Basically everything to do with managing who and
how they use your AWS account.

For our task today, we have to create a user with programmatic access only.
This user that we create isn't going to have console access because we're only
looking at building a dashboard via this user.

Let's start by creating a policy that can be attached to any user. A policy
describes the set of allowed actions on the specified resources. So when a
policy is attached to a user, that user will only be allowed to perform
operations that are outlined in the policy.

For this use case, we just need to be able to check the status of our instances,
so what we're looking for is the `ec2:DescribeInstances` action. The resource
that we want to be able to run this action on **should** be `*`. Unfortunately the
`DescribeInstaces` doesn't allow to be specified for a particular resource so
we _have_ to use the `*` identifier for the resource ([relevant thread](https://forums.aws.amazon.com/thread.jspa?messageID=512129)).

Our policy looks like this: 

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstances"
            ],
            "Resource": [
                "*"
            ]
        }
    ]
}
```

Now, assign this to a user and pick up the user's `Access Key` and `Secret 
Access Key`. This is what we'll use to create the client

---

Let's get back to the client and throw in the credentials we picked up in the
previous step.

From here, we call the `DescribeInstances` function that returns the data
required for me to proceed. A wee bit of LINQ and we're good to go:

```csharp
client
  .DescribeInstances()
  .Reservations
  .SelectMany(r => r.Instances)
  .Select(instance =>
  {
      var isUp = (instance.State.Name == InstanceStateName.Running);
      var upSince = isUp ? (DateTime.Now - instance.LaunchTime) : TimeSpan.FromSeconds(0);
      
      return new 
      { 
          Id              = instance.InstanceId, 
          Name            = instance.KeyName, 
          LastLaunchTime  = instance.LaunchTime,
          UpSince         = upSince,  
          IsUp            = isUp,
          State           = instance.State.Name.Value
      };
  })
```

The really exciting part here, for me is how terse and readable the code is.
There's no explanation of _how_ to do something viz a viz _what_ to do.

With this done, we have the data we need, in the format that we need it. The
next question is how does this magically become a _"dashboard"_?

## LINQPad's `Dump` function

This function truly deserves a section for itself and in my opinion, is what
makes LINQPad _such_ a compelling tool to work with. 

`Dump` is a function that is available on **every** object within you LINQPad
queries. It attempts to show your object in the best way possible.

Let's consider a simple list: 

```cs
new List<string> 
{
  "Ramu",
  "Shamu",
  "Suppandi",
  "Kaalia"
}
```

And call the `.Dump()` function on it like so:

```cs
new List<string> 
{
  "Ramu",
  "Shamu",
  "Suppandi",
  "Kaalia"
}.Dump();
```

And this is what we get: 

![List<string> dump output](/images/_dashboard/simpledump.png)

Neat and concise, right? 

It gets even better. Let's bump it up, consider an object: 

```cs
new
{
  Name = "Ramu",
  Siblings = new [] { "Shamu" },
  Genre = "Comedy",
  Age = 5
}.Dump();
```

We get: 

![Object dump output](/images/_dashboard/objectdump.png)

Wonderfully formatted into key value pairs, and the list within the object is
well formatted too.

Optionally, if we provide the `.Dump()` function a string argument, it will
display it as a title for what is being dumped. Eg:

```cs
new
{
  Name = "Ramu",
  Siblings = new [] { "Shamu" },
  Genre = "Comedy",
  Age = 5
}.Dump("Ramu -- Details");
```

![Dump with title](/images/_dashboard/objectdump-title.png)

It just doesn't stop there. It gets **even more** powerful when used in a context
where we query a DB using LINQ.

Once you've created a _connection_, you're able to drop that connection into a
query and run LINQ queries on tables on your DB. Consider a table called
`cities` that contains the list of cities in your system. To query and fetch
the first row, you would do: 

```cs
Cities.FirstOrDefault()
```

To print them? `.Dump()`! So:

```cs
Cities.FirstOrDefault().Dump();
```

Would give us:

![FirstOrDefault dump output](/images/_dashboard/db-city.png)

Just like an object that we printed before.

But it gets _even_ better! Consider a `Prescription` table that has FK
relations out to 3 other tables -- `Representative`, `Dealer` and `School`. Now
when I pick up the first record in this list with 

```cs
Prescriptions.FirstOrDefault().Dump();
```

![FK dump output](/images/_dashboard/db-fk.png)

It resolves these dependencies and puts them in place! Absolutely wicked!

When you're exploring a Database, The `Dump()` function can be a blessing.
This is one of the reasons why LINQPad forms such a core part of my development
environment.

---

Let's get back to where we left things off, we now have with us a list of
objects. Nothing special about that. 

What to we do to display it? 

![DUMP all the things meme](/images/_dashboard/dump-all-the-thangs.png)

We'll call the `.Dump()` function with `region` as the argument. This should
give us a nicely formatted region wise list of instances and their status.

```cs
client
  .DescribeInstances()
  .Reservations
  .SelectMany(r => r.Instances)
  .Select(instance =>
  {
      var isUp = (instance.State.Name == InstanceStateName.Running);
      var upSince = isUp ? (DateTime.Now - instance.LaunchTime) : TimeSpan.FromSeconds(0);
      
      return new 
      { 
          Id              = instance.InstanceId, 
          Name            = instance.KeyName, 
          LastLaunchTime  = instance.LaunchTime,
          UpSince         = upSince,  
          IsUp            = isUp,
          State           = instance.State.Name.Value
      };
  }).Dump(region.DisplayName);
```

Anddddd:

![Dashboard output](/images/_dashboard/dashboard.png)

Awesome! It **works**, its **maintainable** and to top it off, it is **low tech**. All but
for one question lingers in my mind:

_"You don't expect me to run this all the time to check the results, right?"_

Rest assured, LINQPad has got you covered there as well :) 

## lprun for command line scripting purposes

[lprun](https://www.linqpad.net/lprun.aspx) is LINQPad's offering for running
scripts from the commandline.

Its default output is in the JSON format but we can change that by using the
`format` flag. Setting `-format=html` outputs the **exact same thing** we see
inside LINQPad. This works great for automation. 

Chuck the output into a server periodically via a scheduler task and we're
good to go!

This is what I did and we're more than happy with it, for now. Maybe when we
have more instances, we'll scale up to a better and more **actionable** dashboard
but this works for our current scale.

## Takeaways

- LINQPad is awesome.
    - If you're a C#/F#/VB user, use it!
- We need more tools like LINQPad that foster quick iteration
    - REPLs fall in the same category. So if the language that you're using has a
      REPL, put it to good use
- Don't complicate things, simple is guaranteed to work for a majority of the
  cases -- You can scale when the necessity arises
