---
Title: Dynamic prepared statements in Clojure
Slug: dynamic-prepared-statements-in-clojure
---

# Dynamic prepared statements in Clojure

So just recently, at [Logic Soft](http://www.logicsoft.co.in), we put our first
(small) clojure project into production. It was a small library management
system that was custom developed for one of our existing clients.

At Logic Soft, we are majorly a .NET house but we're beginning to experiment
with different platforms and technologies and felt that this was a small enough
project to give clojure a shot (alongside other experiments we conducted with
it).

This is our first time writing clojure code (at all) so the code is quite naive
and very imperative in nature. Because clojure natively enforces immutability,
that was something that we got for free with the choice of language itself. 

One of the challenges we faced was to dynamically create prepared statements to
query the DB. The requirement for this came from having to generate reports in
the app with a variety of inputs. We didn't want to just append everything into
a single query string since that is a huge risk. Prepared statements are the
way to go.

Also, we just went with using plain old `clojure.java.jdbc` without any DSLs.
We made the decision to use as less libraries as possible and stick to the
absolute essentials writing the major chunk of code ourselves for the sake of
experience.

---

**Disclaimer:** Since this is our first project and my first time as well
writing anything _for_ production in clojure so there might be a **lot** of
mistakes. I am always looking to learn more so please feel free to write in /
tweet me with any constructive criticism and feedback. 

Also this post is _long_ because it has a lot of code in it.

---

Today I'd like to share how we went about constructing these dynamic prepared
statements in clojure. 

### Dependencies

I totally recommend the [lein-try](https://github.com/rkneufeld/lein-try)
plugin by [Ryan Neufeld](http://www.rkn.io/) to quickly try stuff out in the
REPL before using it in the project itself.

These are my dependencies both in `lein-try` format for you to throw into the
command line or in `project.clj` format. 

#### lein-try

```
$ lein try org.clojure/java.jdbc "0.3.0" org.postgresql/postgresql "9.2-1003-jdbc4"
```

#### project.clj

```clojure
:dependencies [;...
               [org.clojure/java.jdbc "0.3.0"]
               [org.postgresql/postgresql "9.2-1003-jdbc4"]]
```

### Querying

Issuing queries on to a db with `clojure.java.jdbc` is really super simple.
Let us start off by `require`ing the `clojure.java.jdbc` namespace in the REPL

```clojure
(require '[clojure.java.jdbc :as j])
```
A `map` is used to define the connection parameters.

```clojure
(def conn {:classname "org.postgresql.Driver"
           :subprotocol "postgresql"
           :subname "//localhost:5432/DB"
           :user "postgres"
           :password "postgres"})
```

With that defined, we can fire a query to get a list of authors from the DB:

```clojure
(j/query conn
         ["select id, trim(name) as name from author order by id"])
```

To fetch a _particular_ author, we use `?` in the query followed by the
parameter in the vector we pass to the `query` method:

```clojure
(j/query db/conn
         ["select id, trim(name) as name from author where id = ?" 1])
```

To insert a new author:

```clojure
(j/insert! conn
           :author
           {:name name}))
```

You can find a lot more great documentation for `clojure.java.jdbc` over on the
[API Reference](http://clojure-doc.org/articles/ecosystem/java_jdbc/home.html)
or on the [community driven documentation
site](http://clojure-doc.org/articles/ecosystem/java_jdbc/home.html)

### Understanding the case at hand 

Let us consider a report that allows us to filter out the books based on the
author, the publisher and parts of the title itself. The base query to get the
columns is quite simple for this: 

**Note:** let us for the sake of not diverging from the topic, not discuss the
_reasoning_ behind the schema

```sql
select  i.isbn, 
        i.title. 
        a.name as author, 
        p.name as publisher

from item as i

inner join author as a 
  on i.author_id = a.id

inner join publisher as p 
  on i.publisher_id = p.id

where i.type = 1
```
If an Author is provided, the query needs to be appended with 

```sql
and a.id = 1
```

If a Publisher is provided, the query needs to be appended with 

```sql
and p.id = 2
```

If parts of a title are provided, say **Art** and **Programming**, then the
query needs to be appended with

```sql
and upper(i.title) like '%ART%' and upper(i.title) like '%PROGRAMMING%'
```
### Preparing the Prepared Statement

We now know that for substitution, we just need to have `?` in the query string
and pass in the arguments appropriately. So considering all the filter are
applied as in the last section, our call to `query` should be something like
this

```clojure
(j/query conn
         ["select i.isbn, 
                  i.title,
                  a.name as author, 
                  p.name as publisher

          from item as i

          inner join author as a 
          on i.author_id = a.id

          inner join publisher as p 
          on i.publisher_id = p.id

          where i.type = 1
                and a.id = ?
                and p.id = ?
                and upper(i.title) like ?
                and upper(i.title) like ?"

          1 2 "%ART%" "%PROGRAMMING%"])
```

How hard can this be? 

Since the core of clojure's data structures are immutable, thinking about how
to do such a thing _dynamically_ took me a bit to figure out.

I took to expressing the construction of the prepared statement in terms of a
map, with a `:query` key to store the base query and a `:params` key to store a
vector of params to be passed in relative to all the `?`s in the `:query`

This went into a `let` block in a function `filter-titles`. All of the prepared
statement construction had to happen with the `let` block since that is one
among the only places where one can _define_ things in clojure.

```clojure
(defn filter-titles [author publisher title-parts]
  (let [ps-params {:query "select i.isbn, 
                                  i.title. 
                                  a.name as author, 
                                  p.name as publisher

                           from item as i

                           inner join author as a 
                           on i.author_id = a.id

                           inner join publisher as p 
                           on i.publisher_id = p.id

                           where i.type = 1"
                   :params []}

        ;...
```

Based on the presence of the filters I had to append a piece of text to the 
query and an argument to the argument list. 

For discussion purposes I've abstracted the part that ensures that `author`,
`publisher` and `title-parts` either has some value or `nil`. This means that
I can use a simple `if` to check for their presence and then append and return
a new `ps-params`.

To take the old `ps-params` and return a new one with the appended where clause
to `:query` and the respective parameter to `:params`, I wrote a function that
takes the old `ps-params`, a `where-clause` to be appended and a
`where-clause-value` to be substituted, if any and returns a new map with the
query and parameter appended

```clojure
(defn add-to-prepared-statement [prepared-statement
                                 where-clause
                                 where-clause-value]
  (let [{:keys [query params]} prepared-statement]
    {:query (str query " " where-clause)
     :params (if where-clause-value
               (conj params where-clause-value)
               params)}))
```

It first extracts the `:query` and `:params` from the given prepared statement

```clojure
(let [{:keys [query params]} prepared-statement]
```

then, return a map with the where clause appended to `:query`

```clojure
{:query (str query " " where-clause)
```
And if there is a `where-clause-value`, `conj` that into the `:params`

```clojure
:params (if where-clause-value
         (conj params where-clause-value)
         params)}
```

With the `add-to-prepared-statement` function in my arsenal, appending to
`ps-params` became easier and the `let` definition of `filter-titles` could be
extended with

```clojure
;...
ps-params (if author
            (add-to-prepared-statement ps-params
                                       "and a.id = ?"
                                       author)
            ps-params)
ps-params (if publisher
            (add-to-prepared-statement ps-params
                                       "and p.id = ?"
                                       publisher)
            ps-params)
;...
```

Okay! So far so good.

At this point though, I hit another road block because based on the amount of
words in `title-part`, I had to dynamically append that many `and
upper(i.title) like ?` clauses into `:query` and that many values into
`:params`.

I could've done this with a loop but decided that I want to do it functionally.

So first, I had to write a function to clean and split the given `title-parts`
string

```clojure
(defn clean-and-split [title-parts]
  (-> title-parts
      (str/trim)
      (str/replace #"\ +" " ")
      (str/split #" ")))
```

For this, I first trim the string, replace multiple spaces with a single space
and split it at the space character to get a list of words. This function uses
another neat clojure feature called the **Thread first macro** (`->`) which I
spoke a little about in my [previous
post](http://www.shrayas.com/a-day-with-clojure.html) about clojure. It is
something that you should definitely [check
out](https://clojuredocs.org/clojure.core/-%3E).

Now that we have a list, we need to generate _that_ many `and upper(i.title)
like ?` strings to be appended into `:query`. 

The way I chose to do this is by using the
[`repeat` function](https://clojuredocs.org/clojure.core/repeat) in clojure
which returns a lazy, or if a length is specified, that many number of
occurrences of whatever is specified. 

```clojure
(let [title-part-splits (clean-and-split title-parts)
      no-of-parts (count title-part-splits)
      title-part-queries (repeat no-of-parts "and upper(i.title) = ?")
      ;...
```

Now that I had a vector containing as many parts I wanted to be appended into the
query, I just used a simple `reduce` to put them all together.

```clojure
(let [title-part-splits (clean-and-split title-parts)
      no-of-parts (count title-part-splits)
      where-clause (reduce #(str %1 " " %2)
                           (repeat no-of-parts 
                                   "and upper(i.title) = ?"))
      ;...
```

before calling `add-to-prepared-statement`, I needed to append and prepend a
`%` character to all the parts of the title and convert them to upper case.
This was simple enough with a map function

```clojure
(let [title-part-splits (clean-and-split title-parts)
      no-of-parts (count title-part-splits)
      where-clause (reduce #(str %1 " " %2)
                           (repeat no-of-parts 
                                   "and upper(i.title) = ?"))
      where-clause-values (map #(str "%" (str/upper-case %) "%")
                               title-part-splits)
      ;...
```

Soon enough I hit the next road block with the `add-to-prepared-statement`
function since I had originally only intended for it to take a **single**
`where-clause-value`. Now I had a vector whose contents needed to be appended
into the `:params` vector.

To do this, I changed the `add-to-prepared-statement` function and checked if
the parameter provided was a single value or a `sequential` and accordingly
either did a `conj` or did an `into`

```clojure
(defn add-to-prepared-statement [prepared-statement 
                                 where-clause 
                                 where-clause-value]
  (let [{:keys [query params]} prepared-statement
        new-query (str query " " where-clause)
        new-params (if where-clause-value
                     (if (sequential? where-clause-value)
                       (into params where-clause-value)
                       (conj params where-clause-value))
                     params)]
    {:query  new-query
     :params new-params}))
```

[This](http://www.brainonfire.net/files/seqs-and-colls/main.html) was a great
guide that helped me understand when to use `coll?`, `sequential?` and the
other collection/sequence comparison functions.

With these modifications, I could now extend the `let` of the original
`filter-titles` function

```clojure
;...
ps-params (if title-parts
            (let [title-part-splits (clean-and-split title-parts)
                  no-of-parts (count title-part-splits)
                  where-clause (reduce #(str %1 " " %2)
                                       (repeat no-of-parts 
                                               "and upper(i.title) = ?"))
                  where-clause-values (map #(str "%" (str/upper-case %) "%")
                                           title-part-splits)]
              (add-to-prepared-statement ps-params
                                         where-clause
                                         where-clause-values))
            ps-params)
;...
```

By the end of this, in `ps-params`, I had all I wanted to create the prepared
statement to pass to jdbc's `query` function. 

```clojure
;...
prepared-statement (-> []
                       (conj (:query ps-params))
                       (into (:params ps-params)))
;...
```

Which really left the body of the `filter-titles` function to be as simple as

```clojure
(j/query conn
         prepared-statement)
```

With this approach, I was able to _dynamically_ create the prepared statement I
wanted without just appending everything into one single query string.

Here is the whole `filter-titles` function for completion sakes

```clojure
(defn filter-titles [author publisher title-parts]
  (let [ps-params {:query "select i.isbn, 
                                  i.title. 
                                  a.name as author, 
                                  p.name as publisher

                           from item as i

                           inner join author as a 
                           on i.author_id = a.id

                           inner join publisher as p 
                           on i.publisher_id = p.id

                           where i.type = 1"
                   :params []}
        ps-params (if author
                    (add-to-prepared-statement ps-params
                                               "and a.id = ?"
                                               author)
                    ps-params)
        ps-params (if publisher
                    (add-to-prepared-statement ps-params
                                               "and p.id = ?"
                                               publisher)
                    ps-params)
        ps-params (if title-parts
                    (let [title-part-splits (clean-and-split title-parts)
                          no-of-parts (count title-part-splits)
                          where-clause (reduce #(str %1 " " %2)
                                               (repeat no-of-parts 
                                                       "and upper(i.title) = ?"))
                          where-clause-values (map #(str "%" (str/upper-case %) "%")
                                                   title-part-splits)]
                      (add-to-prepared-statement ps-params
                                                 where-clause
                                                 where-clause-values))
                    ps-params)
        prepared-statement (-> []
                               (conj (:query ps-params))
                               (into (:params ps-params)))]
    (j/query conn
             prepared-statement)))
```

### Conclusion

This was the approach that I took for the rest of the reports implemented in
the system as well. Because of my history with mutating variables all around
and not _thinking functionally_, it was hard to reason this out initially. But
when it finally [hit](https://twitter.com/shrayasr/status/637242024563834880),
the feeling was priceless :)

My sincerest gratitude goes out [Ramki Sir](http://rkrishnan.org/) and
[Dheeraj](http://codepodu.com/) who helped me with pulling this project
together and seeing it to production. I've come to notice that people in the
Clojure community are very forthcoming and helpful and that means a lot to a
beginner like me with their first project. 

Thank you, everyone! Hope this helped.
