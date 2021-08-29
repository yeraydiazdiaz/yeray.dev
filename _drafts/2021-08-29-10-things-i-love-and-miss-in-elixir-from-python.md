---
layout: post
title: "10 things I love and miss in Elixir coming from Python"
date: 2021-08-29 09:51:19 +0000
categories: elixir
tags: elixir python
short_description: 
keywords: "elixir, python"
---

<div markdown="1" class="sticky">
* TOC
{:toc}
</div>

<div markdown="1" id="text">

I took up [Elixir](https://elixir-lang.org/) about 2 years ago for the first time. I bought a couple of books[^1] about it and just sort of clicked for me. Its focus on developer experience and productivity appealed to me as a Pythonista, and found it refreshing after having tried other functional languages in the past but finding them a bit too academic. So I decided to implement an idea I had for a side project using it.

Fast forward about a year and we have [shoutouts.dev](https://shoutouts.dev): A website for users of OSS to post comments about their favourite projects and for maintainers to showcase them.

Usually I recommend implementing side projects with technologies you're familiar with, since the difficulties while learning can distract from the development of the project. But I felt this was a nice low-risk idea that I wouldn't mind if it didn't quite come to fruition.

Being a long time Pythonista I usually get asked how does Elixir compare with Python. So now that I've gone through a whole journey I wanted to give my perspective of things I love about Elixir but miss from working with Python.

Ready? Here we go!

<!--more-->

## Love: Pattern matching

Yes, I know [Python 3.10 has pattern matching](https://www.python.org/dev/peps/pep-0636/), but it is _everywhere_ in Elixir, so much so that `=` is not assignment, it's pattern match:

```elixir
iex> x = 1
1
iex> 1 = x
1
iex> 2 = x
** (MatchError) no match of right hand side value: 1
```

Virtually all constructs use pattern matching making it super easy and expressive to pick the bits you need from a data structure:

```elixir
iex(1)> [1, a, 3] = [1, 2, 3]
[1, 2, 3]
iex(2)> a
2
```

And add even add [guard clauses](https://hexdocs.pm/elixir/master/patterns-and-guards.html#guards):

```elixir
iex> case {1, 2, 3} do
...>   {1, x, 3} when x > 0 ->
...>     "Will match"
...>   _ ->
...>     "Would match, if guard condition were not satisfied"
...> end
"Will match"
```

Pattern matching on function definitions is incredibly powerful and results in clearer code patterns that are easier to read and maintain. Here's an example from shoutouts where we use pattern matching on a Phoenix LiveView function first for logged in users:

```elixir
  def mount(
        %{"owner" => owner, "name" => name},
        %{"current_user_id" => current_user_id},
        %{assigns: %{live_action: :show}} = socket
      ) do
    # ommitted code
```

And later for anonymous users:

```elixir
  def mount(
        %{"owner" => owner, "name" => name},
        _session,  # note the missing current_user_id
        %{assigns: %{live_action: :show}} = socket
      ) do
      # ommitted code
```

Elixir will attempt to match an incoming request to these functions in order, so if the session map (the second argument) includes a `current_user_id` the first function will be executed, otherwise the second one will. Also note that in the first function Elixir will bind the variables `owner`, `name`, and `current_user_id` based on the contents of each of the maps.

The downside is that you will get many runtime matching exceptions if you're not careful and have a catch-all matching scenario. This is quite common when functions return what's called an "tagged-tuple", i.e. `{:ok, result}` or `{:error, error}`. You may optimistically only match `{:ok, result}` which would raise a matching exception if the function ever returns an error.

It took me a while to fully understand the power of pattern matching and how you can apply it to virtually everywhere in your code as it's quite foreign coming from Python.

## Miss: Explictness

Elixir has a strong meta-programming story and macros are very common both in its standard library external ones, many of which introduce operators or constructs. This can be confusing for someone coming from Python, where you just know what's part of the language and what isn't.

Elixir takes inspiration from Ruby and the [Phoenix framework](https://phoenixframework.org/) is heavily influenced by Ruby on Rails, where it's common to prefer "convention over configuration". For example, you have to adhere to naming conventions to get things to work. This was one of my pet peeves when working on Ruby on Rails and is simply not common in the Python ecosystem where explictness is an axiom of the language.

Elixir is an expression based language, there are no explicit returns, the result of a function or block is the value of the last expression. For example, `if` constructs used to trip me up constantly:

```elixir
iex(1)> hi = "hello!"
"hello!"
iex(2)> if true, do: hi = hi <> " how is it going?"
warning: variable "hi" is unused (if the variable is not meant to be used, prefix it with an underscore)
  iex:2

"hello! how is it going?"
iex(3)> hi
"hello!"
```

That warning is already telling you something but it's not really clear which `hi` it's referring to, and I found it surprising that `hi` was unchanged. But if we remember that `if`s are expressions that evaluate to the last value:

```elixir
iex(4)> hi = if true, do: hi <> " how is it going?"
"hello! how is it going?"
iex(5)> hi
"hello! how is it going?"
```

You do get used to this but it gets a bit complicated when you'd like to return early from a function, something that is trivial in Python.

## Love: Immutability

A common feature amongst functional languages but one that I didn't think I would enjoy so much. It's oddly liberating to just know _for a fact_ that you cannot mutate a data structure. All functions take a data structure and return a new one. Always. No more thinking about what will happen if you pass a dict to a function, or trying to remember if a function returns the structure or if it mutates it:

```elixir
iex(1)> m = %{hello: "world"}
%{hello: "world"}
iex(2)> Map.merge(m, %{foo: "bar"})
%{foo: "bar", hello: "world"}
iex(3)> m
%{hello: "world"}
```

Versus in Python:

```python
>>> m = {"hello": "world"}
>>> n = m.update({"foo": "bar"})
>>> n  # None? so then...
>>> m
{'hello': 'world', 'foo': 'bar'}
```

At first I kept incorrectly trying to call methods on data structures, but after a while I started to appreciate that everything is a function in a module and takes a data structure as a first parameter. Do you want to operate on a map? Call a function in the `Map` module passing your map as the argument. You are guaranteed to get a _different_ map back.

## Miss: For loops

Yep, good ol' `for` loops. Elixir is a functional language and as such it encourages the use of recursion and processing iterables (or enumerables) through functions using the `Enum` module.

```elixir
iex> Enum.map([1, 2, 3], fn x -> x * 2 end)
[2, 4, 6]
```

I didn't find it a problem for simple cases like `map` or `filter` which exist in Python although are less commonly used thanks to comprehensions. But in more complex scenarios you can't rely on the tried-and-tested Python approach of iterating over a collection and having an accumulator variable.

One common example is `Enum.reduce`, even though `reduce` has been in Python for a very long time (famously [banished to `functools` by Guido in Python 3](https://www.artima.com/weblogs/viewpost.jsp?thread=98196)), in Elixir you're pretty much bound to use it and that's usually a struggle. After a few tries you start to get the hang of it, particularly when the end result is not a data structure but number or string:

```elixir
  num_shoutouts =
    Enum.reduce(shoutouts, 0, fn t, acc ->
      if t.pinned, do: acc + 1, else: acc
    end)
```

The most confusing cases happen when trying to use `reduce` to create a new list or map, luckily Elixir has tackled this problem quite elegantly with [comprehensions](https://elixir-lang.org/getting-started/comprehensions.html) but you definitely want to have a solid grasp of `reduce`.

## Love: Pipe operator

Despite the above, `for` loops tend to be where complexity starts to build up quickly. Operations on elements are inlined or conditional branches are added very easily and they're hard to pick apart for a future reader. 

In Elixir, the pipe operator `|>` makes chaining operations a joy. As described previously, Elixir functions generally take a data structure as a first argument and returns a new one. The pipe operator will take the result of the prior function and pass it as the first argument to the next one, for example:

```elixir
def first_name(%User{} = user) do
  user.name
  |> String.split()
  |> List.first()
end
```

This allows you to express complex pipelines clearly and succintly so the reader can understand each operation separately and compose the result. For smaller tasks it's not that big a deal but when there are 4 or more steps you start appreciating how clearly the data flow is described.

## Miss: Ecosystem

One of the best things about Python is the ecosystem. You'd be hard pressed to find any tech, task, or service that does not have a high quality library available, sometimes more than one.

Elixir, being a much younger language, is just not at that stage. You do have options for many things, but the quality and maturity of the libraries vary greatly. That being said, the "core" libraries and frameworks like [Phoenix](https://phoenixframework.org/) and [Ecto](https://hexdocs.pm/ecto/Ecto.html) are just superb. The documentation, the maintainer responsiveness, and quality are really amazing.

Still, that feeling you have in Python where for virtually any task there's probably a library for is just not there yet.

## Love: Tooling

Elixir ships with the Mix tool, which you use for virtually anything, from dependency management to database migrations to file formatting. It's an extensible tool inspired from other young languages like Rust and Elm where there's a strong focus on developer experience.

Unfortunately, nothing quite like that exists for Python, but, as mentioned before, there's a plethora of high quality libraries for many of the common developer tasks. They do require research from the user to understand the problem and decide for one tool or another. In Elixir there's one standard way to do it that ships with the language.

Some other lovely tools in Elixir are the [interactive shell](https://hexdocs.pm/iex/IEx.html) and [Erlang's Observer](https://elixir-lang.org/getting-started/debugging.html#observer), which you can use to connect to any running node and debug the system while it's running which is something I've always wanted to do in Python.

## Miss: Simple releases

In Python shipping to production is fairly straghtforward: you make sure your target platform is running the version Python you want, you install the dependencies, copy the code, and run it. You may add a virtual environment or wrap everything in a Docker container but that's basically it.

In Elixir things get complicated. You _can_ use Docker, but if you can't or don't want to[^2], you need to make a "release". Which is basically a tarball of the Erlang VM along with your compiled code that can be ran "anywhere".

This sounds simple, but it's actually quite hard to set up correctly, because it turns out you cannot build a release for, say, Ubuntu, on a Mac, you need an Ubuntu VM or Docker to build the release. This means you need to write a fairly complex Dockerfile that is not meant for running the code but to create the release and write it out to disk. Finally, you need to copy this to the target machine and run it there.

That being said, there are services like [Heroku](https://elixir-lang.org/blog/2020/09/24/paas-with-elixir-at-Heroku/) or [Gigalixir](https://www.gigalixir.com/) that will make this process much easier, but there's quite a contrast with Python when you want a DIY approach.

## Love: Concurrency

Elixir is based on Erlang which was built with concurrency in mind, in sharp contrast to Python.

Elixir/Erlang have the concept of "processes" which are _not_ OS processes or threads, but more akin to greenlets and are preemptively scheduled by the Erlang VM. These processes are very lightweight, do not share any state with each other, and communicate via messages. Processes just run functions, and functions simply [don't have a colour](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/).

The most common example is making parallel operations and gathering the results is also a breeze with [tasks](https://elixir-lang.org/getting-started/mix-otp/distributed-tasks.html#asyncawait):

```elixir
defp create_projects(repos, current_user) do
  Map.keys(repos)
  |> Stream.map(&String.split(&1, "/"))
  |> Stream.map(fn [owner, name] ->
    Task.async(fn -> Projects.project_info(current_user, owner, name) end)
  end)
  |> Stream.map(&Task.await/1)
  |> Stream.map(fn {:ok, project_info} ->
    Projects.create_project(current_user, Map.from_struct(project_info))
  end)
end
```

On top of that Elixir/Erlang use OTP to orchestrate and define fault-tolerance to processes, by defining strategies for what to do when a process fails, in what is known as "let it crash"[^3].

The result is that Elixir is able to perform tasks in parallel very efficiently and with rock-solid, user-defined, fault tolerance, making it an excellent choice for projects that require high concurrency.

For the developer there's a change in the way of thinking that I felt new coming from Python where typically you have a single process with a single thread. You can certainly build systems like that using threads, processes or coroutines in _asyncio_ in Elixir it feels much more natural.

Personally, I had little need to reach for constructs like GenServers and OTP where this magic really happens, as libraries and frameworks abstract these details from the user and you only need to add them to the application and maybe change a bit of configuration.

That being said, it's good to be familiar with the constructs to understand how things work and to debug problems. For example, [Phoenix LiveViews](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html), one of the most exciting features of Phoenix, are implemented as GenServers, they have the same callbacks and can send and receive messages just like any other process.

This wide range of concurrency constructs in the language feels much more advanced than what Python has to offer at this moment. Even though the new breed of [asyncio libraries](https://github.com/timofurrer/awesome-asyncio) in Python has changed its concurrency story, there's still quite a gap between the two.

## Miss: Not having to compile

Elixir compiles down to Erlang bytecode adding a step in the development cycle that I'm not used to coming from Python.

While the compiler does a great job with its error messages and generally provide helpful suggestions to the user, there's always an overhead when upgrading a library, generating a release, or during development that is simply not there in Python. When working on other compiled languages you generally get benefits like type checking, but Elixir is also dynamically typed so those benefits are not there in the core language[^4].

There's an additional related issue I had when I started working in Elixir: at compile time Elixir will read the configuration files, usually named `dev.exs`, and `prod.exs`, and include them in the executable code; which means that the code will always execute with the same configuration, regardless of where its being ran. 

This is in contrast with the common practice in Python of reading from the environment _at runtime_ and configuring your app from it. There is a recent trend in Elixir to discourage this and prefer environment variables for configuration, but many libraries will still expect compile time configuration resulting in an odd mix that can be quite confusing.

## Bonus: OOP vs functional programming

Elixir is a functional language and as such there is no concept of classes. The closest you'll find in Elixir are [structs](https://elixir-lang.org/getting-started/structs.html) but, of course, they don't have methods and must be interacted with via functions.

OOP was quite ingrained in me and it felt quite foreign to work exclusively with functions and data structures. But eventually I started realising how little state is actually required for most operations and how to design the tasks as a sequence of functions over data.

Of course, there are instances where state needs to be kept and Elixir solves them with a [combination of processes and messages](https://elixir-lang.org/getting-started/mix-otp/agent.html). What are typically long lived objects holding state in Python, in Elixir they're long-running processes that you can communicate with messages.

I also wondered how polymorphism would work in a functional language, but Elixir solves it with [behaviours](https://elixir-lang.org/getting-started/typespecs-and-behaviours.html#behaviours) which define an interface for a _module_, specifying the functions that must be implemented.

Nowadays my Python code has become a lot more functional, although it doesn't feel quite natural, I certainly avoid using classes more than I used to.

## Conclusion

I love Elixir and I love Python.

As with anything there are always trade-offs. I'm quite happy I made the shoutouts.dev journey in Elixir, not to say it was easy but I've reached a point where I feel productive and am able to iterate quickly. It has helped me appreciate things about Python and to think about problems from a different, more functional, perspective.

I encourage you to have a look at what Elixir has to offer and in general to branch out and learn new languages.

And, of course, please check out [shoutouts.dev](https://shoutouts.dev), and let me know if you have any comments or ideas for it in its [repo](https://github.com/yeraydiazdiaz/shoutouts.dev).

Thanks for reading!

---

[^1]: [Elixir in Action](https://www.manning.com/books/elixir-in-action) by Saša Jurić, and [Programming Phoenix](https://pragprog.com/titles/phoenix14/programming-phoenix-1-4/) by Chris McCord, Bruce Tate and José Valim.
[^2]: I chose not to because I wanted to experiment with the [Erlang distributed capabilities](http://erlang.org/doc/reference_manual/distributed.html).
[^3]: This summary doesn't really do justice to concurrency aspect of Elixir/Erlang, Saša Jurić does a much better job at explaining all of this in his excellent talk [The Soul of Erlang and Elixir](https://www.youtube.com/watch?v=JvBT4XBdoUE).
[^4]: You can specify [types](https://elixir-lang.org/getting-started/typespecs-and-behaviours.html) but the compiler will not check them, you need to use a separate tool: [Dialyzer](http://erlang.org/doc/man/dialyzer.html).
</div>
