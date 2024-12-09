---
Title: A day with Clojure
Slug: a-day-with-clojure
---

# A day with Clojure

It has been a hectic month for me at [Logic Soft](http://www.logicsoft.co.in).
I'm working on quite a few [design](https://twitter.com/LogicsoftInd/status/553498441192456192) 
related activities for the upcoming book fairs and other internal office work. In 
doing all of this, most of my programming related activities have been put on hold. 

This was ok at first, but then it kind of started getting to me. I really enjoy
design but then I find it less satisfying than writing programs. 

Yesterday morning, as I was performing my early morning browse-reddit-for-cool-
stuff routine I noticed an inviting link that said [**"Part 6 of building a 
Compojure address book: Deployment"**](http://www.reddit.com/r/Clojure/comments/2sr5fp/part_6_of_building_a_compojure_address_book/) 
on [/r/clojure](http://www.reddit.com/r/clojure). From my previous experience, I
knew Compojure had something to do with web apps so I opened this up and checked 
if all the other parts from 1 through 5 were there for completeness sake. After 
a quick browse, I decided that I'd spend my day with this.

I've dabbled [on](https://github.com/shrayasr/learning-clojure) and 
[off](https://github.com/shrayasr/githuj) with Clojure but i've never really had
the time to go through with any of them. This seemed small enough for me to 
complete within a day and substantial enough for me to learn some nice Clojure
concepts. Clojure to me has been really fascinating for a variety of reasons 
ranging from _It is a lisp_ to _Immutability_ to _Rich Hickey_'s talks; but that
is a post for another day. 

This post details some of my learnings, both good and bad in and about Clojure 
as I followed this tutorial.  You can find my interpretation of the tutorial code 
[here](https://github.com/shrayasr/learning-clojure-address-book).

### 1. Hiccup

[Part 2](http://www.jarrodctaylor.com/posts/Compojure-Address-Book-Part-2/) of the
tutorial introduced HTML templating using [Hiccup](https://github.com/weavejester/hiccup)
and I was marvelling at the simplicity of representing HTML within clojure. I have 
heard about the ease of writing [DSLs](http://en.wikipedia.org/wiki/Domain-specific_language)
with Clojure but this just set things into perspective. Here is the first piece
of Hiccup code I encountered:

```clojure
(defn common-layout [& body]
  (html5
    [:head
      [:title "Address Book"]
      (include-css "/css/address-book.css")]
    [:body
      [:h1#content-title "Address Book"]
      body]))
```

There is a lot of readability and a great level of expressiveness in this code. 
Jarrod uses much complex HTML structures in later parts of his tutorial but everything 
falls under the same level of readability and expressiveness as this one. 

### 2. Tests

I find writing tests for a web app non trivial since we have to _mock_ the 
request but a combination of the [Midje](https://github.com/marick/Midje) 
framework and [ring request mocking](https://github.com/weavejester/ring-mock) 
provides a really nice platform to work with to write tests. This is one of the 
more complex tests that we wrote in the project to test the delete functionality:

```clojure
(fact "Test delete"
   (query/insert-contact<! {:name "A"
                             :phone "1"
                             :email "a@x.com"})
   (query/insert-contact<! {:name "B"
                             :phone "2"
                             :email "b@x.com"})
   (count (query/all-contacts)) => 2
   (let [response (app (mock/request :post "/delete/1"))]
     (:status response) => 302
     (count (query/all-contacts)) => 1
     (first (query/all-contacts)) => {:id 2
                                      :name "B"
                                      :phone "2"
                                      :email "b@x.com"}))))
```

The way that the entire pain of mocking a request is taken care of by 
`(app (mock/request :post "/delete/1"))` and nothing more was something I found,
calm. It is really non trivial to read this code and understand what this test is 
trying to do as opposed to having lines and lines of monkey patching to mock a 
request. 

### 3. Atoms

In [part 3](http://www.jarrodctaylor.com/posts/Compojure-Address-Book-Part-3/) of
the tutorial, Jarrod uses [Atoms](http://clojure.org/atoms) to manage the list of
contacts in memory. I was under the impression that Atoms in Clojure are the same 
as in Scheme but I was mistaken. Atoms are a way of managing shared, synchronous 
and independent state in Clojure. The documentation on Atoms in Clojure is a very 
nice read to understand how and when to use Atoms

### 4. Thread last macro

The one thing that I've heard most lisp-ers talk about is the power of macros.
Today I was witness to the sheer power of the concept of macros and in particular,
Clojure's [Thread last macro](http://clojuredocs.org/clojure.core/-%3E%3E). In
[part 3](http://www.jarrodctaylor.com/posts/Compojure-Address-Book-Part-3/) of
the tutorial, Jarrod writes a function `next-id` to calculate the next id to be
used when a new contact is being created and the code is like this:

```clojure
(defn next-id []
  (->>
    @contacts
    (map :id)
    (apply max)
    (+ 1)))
```

When I understood what this does, I just put my hand on my mouth and smiled like
a kid getting a new toy. This, was **beautiful**. Let me try and explain this. 

The thread last macro takes the result of an expression and inserts it in as the 
last item in the next form. This means that the result of `@contacts` is inserted
as the last item to `(map :id)`. Further, the result of that is inserted as the
last item to `(apply max)` and so on. The expression, overall would be something
like this: 

```clojure
(+ 1 (apply max (map :id @contacts)))
```

Using the thread last macro makes this much easier to read. It feels like
reading a set of steps and that boosts the understandability of the code. Here 
is the output at each step:

1. `@contacts` [dereferences](https://clojuredocs.org/clojure.core/deref) the 
`contacts` atom and gives us `[{:id 1 :name "A"} {:id 2 :name "B"}]`. This output 
is inserted as the last item to the map function

2.  The [`map` function](https://clojuredocs.org/clojure.core/map) takes input 
from Step 1 and becomes like so `(map :id [{:id 1 :name "A"} {:id 2 :name "B"}])`. 
This will apply the `:id` function on to each of the map in the list and return 
an array of the id's in the map VIZ `[1 2]`.  This output is inserted as the last 
item to the apply function

3. The [`apply` function](https://clojuredocs.org/clojure.core/apply) applies a 
function to a given list of arguments. In this case we apply the `max` function to 
the input from step 2. So `(apply max [1 2])` becomes `(max 1 2)` and gives us 
`3` which is inserted as the last item to the + function.

4. The `+` function adds 1 to the result in step 3 and returns it

The way all this was expressed in such a short and sweet manner using the Thread 
last macro really left me in awe.

### 5. Yesql

[Yesql](https://github.com/krisajenkins/yesql) is a library for interacting with
databases with Clojure in a rather different way. From what I remember with my
work in other languages, I've written SQL queries that I make from within the
program in 2 ways: 

1. Using ORMs like [SQLAlchemy](http://www.sqlalchemy.org/)
2. Plain vanilla SQL inside language specific strings

Yesql brings a new thought process to the table where we keep our sql as sql 
within `.sql` files. You then use the `defqueries` function within `Yesql.core`
to define queries based on the queries in the `.sql` files. 

To illustrate, [Part 4](http://www.jarrodctaylor.com/posts/Compojure-Address-Book-Part-4/)
of the tutorial talks about replacing the `@contacts` atom with an actual db.
Consider the query to select the list of all contacts from the db. It would be:

```sql
-- name: all-contacts
-- Returns a lits of all the contacts 
SELECT name, phone, email from CONTACTS
```

once you use `Yesql.core`'s `defqueries` on a file containing the above sql, it
will generate a `all-contacts` function that we can use for firing the query on
the db. Here is how we use the [`defqueries`](https://github.com/krisajenkins/yesql#one-file-many-queries) 
method to define a query out of this sql file:

```clojure
(defqueries "path/to/sql/file.sql" ...)
```

What this does, is it looks through `file.sql` and within the module, creates
methods taking the name from the comment above the query (`-- name: all-contacts`). 
Further, this module can be imported from other places and used like this:

```clojure
(:require path.to.module :as query)
(print (query/all-contacts))
```

This would fire the corresponding SQL from the file on the DB.

I found this a rather interesting coming from a world where queries were written 
in rather primitive formats.

### 5. General readability of Clojure and Lisps

I've always been a giant fan of Lisps. I find the [S-Expression](http://en.wikipedia.org/wiki/S-expression) 
notation very readable and far more understandable after a point when compared to 
the infix notation. 

This was my first experiment with writing **proper** Clojure. I.e. Clojure that
actually makes an application run and I should say that my notion of increased
readability with Lisps has been strengthened. There were very few parts on the
tutorial that I got stuck because of me **not** understanding the code. 

This I think is the power that Lisps come with. Once you get a sense of 
S-Expressions, they become a breeze to read.

### 6. Compile time

One of the things that I felt that would be an instant discouragement to newcomers
is the amount of time it takes to run a Clojure project. At the end of [Part 1](
http://www.jarrodctaylor.com/posts/Compojure-Address-Book-Part-1/) of the tutorial, 
we had a very simple `GET` and `POST` based web server and some tests behind it. 
Running `lein midje :autotest` for the tests took a noticable 10s _at the least_ 
and running `lein ring server` to bring up the server took more time than that. 

This I felt was an instant discouragement because it breaks the flow in your
head and doesn't allow for easy experimentation and rapid learning.

---

All in all, I had a great day learning Clojure and I really have to thank 
[Jarrod](https://twitter.com/JarrodCTaylor) for his amazing 5 part tutorial which
is what made all this happen. Please feel free to point out any mistakes I might 
have made in this post, for I am still learning my ways around this beautiful
language and its concepts.
