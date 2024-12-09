---
Title: Express solutions elegantly using your programming language
Slug: express-solutions-elegantly-using-your-prog-lang
---

# Express solutions elegantly using your programming language

## Introduction

The post features a case study of a real life problem and the C# programming
language. It explains how we went about trying to best _express_ the solution
within the bounds of the C# language.

Although we use the C# programming language here,the ideas shared in this post
aren't specific to it and can be applied to any high level language you are
working with.

## Problem

Recently, for a customer requirement we had to work with a document that
has both metadata and data. This doc needed to be parsed and interpreted.
Metadata is enclosed within `#` symbols and the associated value is provided
just after it.

Example:

```
...
#BookName#Foobar
#ChapterName#Adventures of Foo
#SectionName#Introducing Foo
...
```

Tags allow for identifying various aspects of the document like chapters,
sections, book names amongst others.

The task was to parse out the document, identify sections and extract the
necessary information.

## Solution

The first step is to identify the paragraphs that contain tags that are of 
importance to us. Following this, we need to be able to parse out the data
within these tags.

Finally we put the metadata together with the data and return a
`parsedDocument`.

### Part 1

A loop that iterates paragraph by paragraph must now check the contents of the
para for respective tags. For this, the tags have to be declared upfront. One
way to do this is by declaring simple constants in the class: 

```cs
private const string chapterNameTag = "#ChapterName#";
private const string sectionNameTag = "#SectionName#";
```

So the check block becomes:

```cs
if (text.Has(chapterNameTag))
{
  // ...
}
```

(Note: `Has` is a simple [C# extension method](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods)
on top of the `StartsWith` function to do the check case insensitively.)

Simple constants work but I could do better, so I pulled out all the tags and
moved them to a nested private class as public constants, so:

```cs
public class Parser
{
  private class Tags
  {
    public const string ClassName = "#ClassName";
    public const string SectionName = "#SectionName#";
    // ...
  }
}
```

The checks now become: 

```cs
if (text.Has(Tags.ClassName))
{
  // ...
}
```

This feels better than plain constants declared at the top. It feels more
_intuitive_ that new tags have a designated _place_ to be declared viz a viz being
declared as just constants in the class

### Part 2

Once we have identified the tag, the next step is to extract the `<name>`
portion from it. In our case, we'd like to get out the **Introduction to Foo** from
the `#SectionName#` tag. The simplest and initial approach to this was:

```cs
if (text.Has(Tags.SectionName))
{
  parsedDocument.SectionName = ExtractTextFromTag(text, Tags.SectionName);
}
```

This works, but `ExtractTextFromTag` is just a lone standing function, which
kind of didn't _sit_ for me (Pun intended). I thought it belongs better inside
the `Tags` class.

To satisfy my want, here's what I _could_ do:

```cs
if (text.Has(Tags.SectionName))
{
  parsedDocument.SectionName = Tags.ExtractFromText(text, Tags.SectionName));
}
```

This solves it, sure. But I still felt I could do better because there is a
repetition of `Tags.SectionName`. I wanted to be able to call a method on
`Tags.SectionName` to parse it out. Think of it like calling `DateTime.Parse`.

This is what I _wanted_:

```
parsedDocument.SectionName = Tags.SectionName.Parse(text);
```

Hard to achieve, I thought since we've declared `Tags.SectionName` as a string. 

The first thought was to add an **extension** method on the string class (like
we did for the `Has` function). But this is a dead end since it would be
available to **all** instances of the string class.

Let's restate the problem we're facing:

_"I wanted a way to describe `Tags.SectionName` in a way that would allow me to
use it as a **regular string** for comparison purposes but still be a **class**
where I could declare the `Parse` functionality."_

With this thought in mind, a bit of googling led me to the solution:

C# allows to define methods in a class using the [`implicit`
keyword](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/implicit).
Doing this allows you to define functionality for how the object should behave
if it is treated as another type. Think of it like explicit type casting (a la
`OtherType otherType = (OtherType)myType`) but just implicit.

With that in mind, I defined a `Tag` class like so:

(Note: Error handling avoided for the same of brevity and discussion)

```cs
public class Tag
{
    private string _value;
    
    public Tag(string tag) 
        => _value = tag;

    public string Parse(string textWithTag)
        => Regex.Replace(textWithTag, _value, "", RegexOptions.IgnoreCase);

    public static implicit operator string(Tag t)
        => t._value;
}
```

Internally a `Tag` is just a value, so we mandate a tag to contain a value by
accepting it in its constructor.

This is followed by the definition of the `Parse` function. It takes the text
with the tag (say `#SectionName#Introduction to Foo`) and strips out the
`#SectionName#` part ignoring the case. We use `System.Text`'s `Regex` class to
do this for us. Simple and straightforward.

What follows next is the usage of the `implicit operator string` which tells
C#: _"When this class is used as a `string`, return the `value` property"_.

**SUUUPER neat. WOO!**

Now this allowed me to redefine the original `Tags` class like so:

```cs
public class Parser
{
  private class Tags
  {
    public static readonly Tag ClassName = new Tag("#ClassName");
    public static readonly Tag SectionName = new Tag("#SectionName#");
    // ...
  }
}
```

Which means that I can now use this as **both** a string, and an instance on which
I have a parse method. So my `if` condition and the parse block becomes: 

```cs
if (text.Has(Tags.SectionName))
{
    parsedDocument.SectionName = Tags.SectionName.Parse(text);
}
```

This made me happy so I stopped right here.

## Conclusion

I'm quite happy that everything is well contained and works together. This
however isn't the approach I took for the program since I think its overkill
for the particular scenario in the application. But it was a good opportunity
to push myself to learn better ways to express a solution for such a problem.

## Take aways

- When you think something isn't possible with the language, dig deeper. If you
  can think about it, chances are that someone already has and it has been 
  solved
- Learn the technical terms (In this case, `implicit type conversion C#` gave me
  what I was looking for). It becomes that much easier to narrow down your
  search when looking for answers
- Don't settle for what _just works_, ask yourself "what is the **best** way to
  represent my problem" and try to achieve that

## Credits

Major props to [Bargava](https://twitter.com/bargava) for helping with proof
reading the blog and [VK](https://twitter.com/argvk) for helping bounce ideas
off of on a lazy Sunday morning.

