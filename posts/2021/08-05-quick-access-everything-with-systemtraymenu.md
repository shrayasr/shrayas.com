---
title: Quick access with SystemTrayMenu
---

# Quick access with SystemTrayMenu

# Problem

Being head of engineering at a firm means having to juggle a lot of context. And in this quest, I am always looking for ways to optimize my workflow. I can (greatly) simplify all of the work I do into these categories: 

1. Talking to folks -- Zoom
   - Usually via pre defined links - My personal link / Standup link / My founder's personal link
2. My own work -- Visual Studio
   - Open a project in Visual Studio from my local directory of projects
3. Collaboration -- GitHub
   - Open a project in GitHub to log an issue / review a pull request

A tool that I already use quite extensively is [Key Pirinha](https://keypirinha.com/) which is the Alfred/Spotlight/Launchy equivalent for windows and is blazing fast. I could do all of the above tasks in it in some fashion but it doesn't do much justice and needs a lot of setup.

I was looking for a way to organize all of this together and have one place to access them from - without me having to code anything for it. In other words, I was looking for a better start menu - one that I could entirely control myself.

# My Solution

In my search for a solution, GitHub one day suggested a project on the home page - [SystemTrayMenu](https://github.com/Hofknecht/SystemTrayMenu) by [Markus Hofknecht](https://twitter.com/MarkusHofknecht). I was intrigued just looking at the name and I checked it out. 

> SystemTrayMenu is a free, open-source start menu alternative for Microsoft Windows. It offers a clear, personalized menu in the system tray. Files, links and folders are organized in several levels as drop-down menus.

![](https://user-images.githubusercontent.com/52528841/90009201-ee0c9680-dc9d-11ea-9b8a-b34108152f9b.gif)

I was immediately sold! This seemed like the right tool for me to use to build out my requirement on.

## Projects

To get a feel for things, I begun by wanting to solve the hardest of my problems - #2 - Opening a project in VS from my local directory. I just gave my `~/work/code` directory to STM and was able to on demand open up any folder and its associated project file in my IDE - This improved my productivity greatly! The search is pretty great too, its not fuzzy but then it solves the problem well enough.

With this confidence, I wanted to take it forward and apply it to the other parts of my workflow. The biggest problem in wanting to solve all 3 problems is that I **did not** want to rearrange any part of my folder structure to support usage via STM.

STM has a fantastic feature of following links. This means that I can make a link called `projects` to my `~/work/code` folder in a `~/STM` folder and start STM pointing to `~/STM` and it will follow `projects -> ~/work/code` and allow me to browse it. This makes it super easy for anyone to set up their "quick access" folders by just creating shortcuts. 

So with that in place, my STM folder started to take shape

```
STM/
└── projects -> ~/work/code
```

## Zoom links

Since zoom links are fundamentally just URLs, I can just create shortcuts to them in another directory. To create a URL shortcut on windows, we have to create a file and do 2 things 

1. Use the extension `.url` 

2. Use the following content in the file

   ```
   [InternetShortcut]
   URL=http://shrayas.com
   ```

So with this, I then created the required zoom URL links and put them into my STM folder under the subdirectory `zoom`

```
STM/
├── projects -> ~/work/code
└── zoom/
    ├── standup.url
    ├── my.url
    └── name1.url
```

## GitHub 

This was the part that took me some time. On a daily basis, I have to review code in / file issues on / share links to a quite a sum of projects. And for that it is important that I can open the repo on GitHub quickly. 

I am not a fan of GitHub's search. I think its slow and many a time, doesn't throw up the result that I am expecting. Because of this, a while ago .. I wrote a small program for myself to help me with this and called it GitHub Browse 

![](/images/_STM/ghbrowse1.png)

![](/images/_STM/ghbrowse2.png)

It was a quick way for me to open any project that I was interested in. The data was stored as a JSON file that I could refresh using my Personal Access Token from GitHub. It worked fine. 

Since I was moving my workflow to STM, I thought it be wise to consolidate it over there instead of maintaining a separate program for this. So I modified the program to instead spit out `.url` files linking to all the repositories into a `gh` folder under the `STM` folder. 

```
STM/
├── projects -> ~/work/code
├── zoom/
│   ├── standup.url
│   ├── my.url
│   └── name1.url
└── gh/
    ├── project1.url
    ├── project2.url
    └── project3.url
```

An unfortunate downside of this was that I lost the ability to directly go to issues or PRs or to just copy the link.. but those are usually just a click away so I was OK with making that compromise. This meant that I had one less piece of code to maintain :) 

## Cleaning up

I would like to use the same STM folder for some personal stuff too, so I just cleaned things up my moving my projects and the GitHub links into a `work` folder 

```
STM/
├── work/
│   ├── projects -> ~/work/code
│   └── gh/
│       ├── project1.url
│       ├── project2.url
│       └── project3.url
└── zoom/
    ├── standup.url
    ├── my.url
    └── name1.url
```

And I was done! It just works​!

![](/images/_STM/stm.gif)

# Conclusion

If you're looking for a project that will help you to optimize your workflow and allow you to quickly access URLs or folders on your file system, do consider [SystemTrayMenu](https://github.com/Hofknecht/SystemTrayMenu). It is a fantastic tool to add to your arsenal. 

Always scratch your itch. It can lead down wonderful paths of exploration and learning. Being Head of Engineering means I don't spend too much time writing code. This experiment of mine allowed me to jump back into my coding roots and figure things out - I had a lot of fun! :) I even ended up automating the process of creating the `STM` directory into a nice little C# DSL that I ended up calling *Quick* and looks like this: 

```csharp
new QuickBase(
  new QuickFolder("work", 
    new QuickFolder("gh", new QuickGithubDump(GITHUB_ACCESS_TOKEN)),
    new QuickFolderShortcut("projects", @"D:\work\code")
  ),
  new QuickFolder("zoom", 
    new QuickURLShortcut("standup", "https://zoom.us/j/..."),
    new QuickURLShortcut("me", "https://zoom.us/j/...")
  )
);

quick.Execute(@"D:\quick");
```

With that, I'm signing off. Take care, stay safe, get vaccinated. 
