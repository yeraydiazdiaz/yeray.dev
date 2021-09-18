---
layout: post
title: "10 things I love and miss in Elixir coming from Python"
date: 2022-03-16 19:28:19 +0000
categories: elixir
tags: elixir python
image: elixir_python.jpg
short_description: 10 things I love and miss in Elixir from Python
keywords: "elixir, python"
---

<div markdown="1" class="sticky">
* TOC
{:toc}
</div>

<div markdown="1" id="text">

I took up [Elixir](https://elixir-lang.org/) about 3 years ago for the first time. I bought a couple of books[^1] about it and just sort of clicked for me. Its focus on developer experience and productivity appealed to me, and found it refreshing after having tried other functional languages in the past but finding them a bit too academic. So I decided to implement an idea I had for a side project using it.

Fast forward about a year and we have [shoutouts.dev](https://shoutouts.dev): A website for users of Open Source Software to post messages of gratitude about their favourite projects.

Being a long time Pythonista I usually get asked how does Elixir compare with Python. So now that I've gone through the whole journey I wanted to give my perspective on things I love about Elixir but miss from working with Python.

Ready? Here we go!

<!--more-->

## 1. Love: Pattern matching

Yes, I know [Python 3.10 has pattern matching](https://www.python.org/dev/peps/pep-0636/), but it is _everywhere_ in Elixir, so much so that `=` is not assignment, it's pattern match:

```elixir
iex> x = 1
1
iex> 1 = x
1
iex> 2 = x
** (MatchError) no match of right hand side value: 1
```

Virtually all constructs use pattern matching making it incredibly easy and expressive to pick the bits you need from a data structure:

```elixir
iex(1)> [1, a, 3] = [1, 2, 3]
[1, 2, 3]
iex(2)> a
2
```

And even add [guard clauses](https://hexdocs.pm/elixir/master/patterns-and-guards.html#guards):

```elixir
iex> case {1, 2, 3} do
...>   {1, x, 3} when x > 0 ->
...>     "Will match"
...>   _ ->
...>     "Would match, if guard condition were not satisfied"
...> end
"Will match"
```

Pattern matching on function signatures is incredibly powerful and results in clearer code patterns that are easier to read and maintain. Here's an example from shoutouts where we use pattern matching on a Phoenix LiveView function first for logged in users:

```elixir
  def mount(
        %{"owner" => owner, "name" => name},
        %{"current_user_id" => current_user_id},
        %{assigns: %{live_action: :show}} = socket
      ) do
    # omitted code
```

And later for anonymous users:

```elixir
  def mount(
        %{"owner" => owner, "name" => name},
        _session,  # note the missing current_user_id
        %{assigns: %{live_action: :show}} = socket
      ) do
      # omitted code
```

Elixir will attempt to match an incoming request to these functions in order, so if the `session` map (the second argument) includes a `current_user_id` the first function will be executed, otherwise the second one will. Elixir will also bind the variables `owner`, `name`, and `current_user_id` based on the contents of each of the maps.

The downside is that you will get many runtime matching exceptions if you're not careful and include a catch-all matching scenario. This is quite common when functions return a "tagged-tuple", `{:ok, result}` or `{:error, error}`. You may optimistically only match `{:ok, result}` which would raise a matching exception if the function ever returns an error.

It took me a while to fully understand the power of pattern matching and how you can apply it to virtually everywhere in your code as it's quite foreign coming from Python.

## 2. Miss: Exceptions

Python _loves_ exceptions, and I know there's a saying about not using exceptions for flow of control, but that's just what you do in Python so it quickly becomes second nature.

Elixir does have exceptions, but the convention is to return tagged-tuples unless something completely unexpected happens. As a result you barely write any exception handling code in Elixir and rely on pattern matching against the return values.

In Python it's quite common to define exceptions that don't necessarily represent an error, the developer can then choose if and where to handle them. For example, Django's [`ObjectDoesNotExist`](https://docs.djangoproject.com/en/3.2/ref/exceptions/#objectdoesnotexist) is raised when a query does not return a result, you can then choose whether to catch the exception and performa different task or let it bubble and produce a 404, which can be quite handy.

In Elixir you can also [define, raise, and catch (rescue) exceptions](https://elixir-lang.org/getting-started/try-catch-and-rescue.html). But generally libraries expose two flavours of the same function, where the version that may raise an exception is suffixed with a `!`, e.g. [`File.cd!`](https://hexdocs.pm/elixir/File.html#cd!/1), whereas its counterpart, [`File.cd`](https://hexdocs.pm/elixir/File.html#cd/1) will return a tagged tuple. I find this convention quite useful to allow the developer to choose based on the likelihood of the task failing and the consequences if it does, but I rarely ever write exception handling code.

The drawback of using tagged tuples is that you have to make sure they are handled properly across the call stack, whereas in Python you can ignore whole sections because you know the exception will bubble and you can handle it wherever is more convenient. This is not a massive problem but I do miss having that resource from time to time.

## 3. Love: Immutability

A common feature amongst functional languages but one that I didn't think I would enjoy so much. It's oddly liberating to just know _for a fact_ that you cannot mutate a data structure.

All functions take a data structure and return a new one. Always.

No more thinking about what will happen if you pass a dict to a function, or trying to remember if a function returns the structure or if it mutates it:

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

At first I kept incorrectly trying to call methods on data structures, but after a while I started to appreciate that all functions in a module and take a data structure as a first parameter and return a new one. Do you want to operate on a map? Call a function in the `Map` module passing your map as the argument. You are guaranteed to get a _different_ map back.

## 4. Miss: For loops

Yep, good ol' `for` loops. Elixir is a functional language and as such it encourages the use of recursion and processing iterables (or enumerables) through functions using the `Enum` module.

```elixir
iex> Enum.map([1, 2, 3], fn x -> x * 2 end)
[2, 4, 6]
```

I didn't find it a problem for simple cases like `map` or `filter` which exist in Python although are less common thanks to comprehensions. But in more complex scenarios you can't rely on the tried-and-tested approach of iterating over a collection and having an accumulator variable.

One common example is `Enum.reduce`, even though `reduce` has been in Python for a very long time (famously [banished to `functools` by Guido in Python 3](https://www.artima.com/weblogs/viewpost.jsp?thread=98196)), in Elixir you're pretty much bound to use it and that's usually a struggle. After a few tries you start to get the hang of it, particularly when the end result is not a data structure but number or string:

```elixir
  num_shoutouts =
    Enum.reduce(shoutouts, 0, fn t, acc ->
      if t.pinned, do: acc + 1, else: acc
    end)
```

The most confusing cases happen when trying to use `reduce` to create a new list or map, luckily Elixir has tackled this problem quite elegantly with [comprehensions](https://elixir-lang.org/getting-started/comprehensions.html) but you definitely want to have a solid grasp of `reduce`.

## 5. Love: Pipe operator

Despite the above, `for` loops is where complexity tends to build up quickly. What starts as a few lines quickly starts growing and conditional branches start to pile up, making it hard to pick apart for a future reader.

In Elixir, the pipe operator `|>` makes chaining operations a joy. As described previously, Elixir functions generally take a data structure as a first argument and returns a new one. The pipe operator will take the result of the prior function and pass it as the first argument to the next one, for example:

```elixir
def first_name(%User{} = user) do
  user.name
  |> String.split()
  |> List.first()
end
```

This allows you to express complex pipelines clearly and succinctly so the reader can understand each operation separately and compose the result. For smaller tasks it's not that big a deal but when there are 4 or more steps you start appreciating how clearly the data flow is described.

## 6. Miss: Ecosystem

One of the best things about Python is the ecosystem. You'd be hard pressed to find any tech, task, or service that does not have a high quality library available, sometimes more than one.

Elixir, being a much younger language, is just not at that stage. You do have options for many things, but the quality and maturity of the libraries vary greatly. That being said, the "core" libraries and frameworks like [Phoenix](https://phoenixframework.org/) and [Ecto](https://hexdocs.pm/ecto/Ecto.html) are just superb. The documentation, the maintainer responsiveness, and code quality are really amazing.

Still, that feeling you have in Python where for virtually any task there's probably a library for is just not there yet.

## 7. Love: Tooling

Elixir ships with the Mix tool, which you use for virtually anything, from dependency management to database migrations to file formatting. It's an extensible tool inspired from other young languages with a strong focus on developer experience like Rust and Elm.

Unfortunately, nothing quite like that exists for Python, and even though there's a plethora of high quality libraries for many of the common developer tasks, they do require research from the user to understand the problem and decide for one tool or another. In Elixir there's one standard way to do it that ships with the language.

Some other lovely tools in Elixir are the [interactive shell](https://hexdocs.pm/iex/IEx.html) and [Erlang's Observer](https://elixir-lang.org/getting-started/debugging.html#observer), which you can use to connect to any running node and debug the system while it's running which is something I've always wanted to do in Python.

## 8. Miss: Simple releases

In Python shipping to production is fairly straightforward: you make sure your target platform is running the version Python you want, you install the dependencies, copy the code, and run it. You may add a virtual environment or wrap everything in a Docker container but that's basically it.

In Elixir things get complicated. You _can_ use Docker, but if you can't or don't want to[^2], you need to make a "release". Which is basically a tarball of the Erlang VM along with your compiled code that can be executed.

This sounds simple enough, but it's actually quite hard to set up correctly, because it turns out you cannot build a release for, say, Ubuntu, on a Mac, you need an Ubuntu VM or Docker to build the release. This means you need to write a fairly complex Dockerfile that is not meant for running the code but to create the release and write it out to disk. Finally, you need to copy this to the target machine and run it there.

That being said, there are services like [fly.io](https://fly.io/), [Gigalixir](https://www.gigalixir.com/), or [Heroku](https://elixir-lang.org/blog/2020/09/24/paas-with-elixir-at-Heroku/) that will make this process much easier, but there's quite a contrast with Python when you want a DIY approach.

## 9. Love: Concurrency

Elixir is based on Erlang which was built with concurrency in mind, in sharp contrast to Python.

Elixir/Erlang have the concept of "processes" which are _not_ OS processes or threads, but more akin to green threads and are preemptively scheduled by the Erlang VM. These processes are very lightweight, do not share any state with each other, and communicate via messages. Processes just run functions, and functions simply [don't have a colour](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/).

Processes are quite simple to work with but Elixir has excellent abstractions for working with them. The most common example is making parallel operations and gathering the results is also a breeze with [tasks](https://elixir-lang.org/getting-started/mix-otp/distributed-tasks.html#asyncawait):

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

The result is that Elixir is able to perform tasks in parallel very efficiently and with rock-solid, user-defined, fault tolerance. Making it an excellent choice for projects that require high concurrency.

This requires a change in the way of thinking coming from Python, where typically you have a single process with a single thread. You can certainly build complex systems using threads, processes or coroutines in _asyncio_, but in Elixir it feels much more natural and powerful.

Personally, I had little need to reach for constructs like GenServers and OTP where this magic really happens, as libraries and frameworks abstract these details from the user and you only need to add them to the application and maybe change a bit of configuration.

That being said, it's good to be familiar with the constructs to understand how things work and to debug problems. For example, [Phoenix LiveViews](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html), one of the most exciting features of Phoenix, are implemented as GenServers, they have the same callbacks and can send and receive messages just like any other process.

This wide range of concurrency constructs in the language feels much more advanced than what Python has to offer at this moment. Even though the new breed of [asyncio libraries](https://github.com/timofurrer/awesome-asyncio) in Python changed its concurrency story, there's still quite a gap between the two.

## 10. Miss: Not having to compile

Elixir compiles down to Erlang bytecode adding a step in the development cycle that I was not used to coming from Python.

While the compiler does a great job with its error messages and generally provide helpful suggestions to the user, there's always an overhead when upgrading a library, generating a release, or during development that is simply not there in Python. 

When working on other compiled languages a big benefit of the compilation step is type checking, but Elixir, like Python, is dynamically typed, so those benefits are not there in the core language. You can specify [types](https://elixir-lang.org/getting-started/typespecs-and-behaviours.html) but the compiler will not check them, you need to use a separate tool [Dialyzer](http://erlang.org/doc/man/dialyzer.html), much like you would use [mypy](https://mypy.readthedocs.io/en/stable/) in Python.

There's an additional related issue I had when I started working in Elixir: at compile time Elixir will read the configuration files, usually named `dev.exs`, and `prod.exs`, and include them in the executable code; which means the compiled code will always execute with the same configuration, regardless of where its being ran.

This is in contrast with the common practice in Python of reading from the environment _at runtime_ and configuring your app from it. There is a recent trend in Elixir to discourage this and prefer environment variables or files read at runtime for configuration, but many libraries will still expect compile time configuration resulting in an odd mix that can be quite confusing.

## Bonus: OOP vs functional programming

Elixir is a functional language and as such there is no concept of classes. The closest you'll find in Elixir are [structs](https://elixir-lang.org/getting-started/structs.html) but, of course, they don't have methods and must be interacted with via functions.

OOP was quite ingrained in me and it felt quite foreign to work exclusively with functions and data structures. But eventually I started realising how little state is actually required for most operations and how to design the tasks as a sequence of functions over data.

Of course, there are instances where state needs to be kept and Elixir solves them with a [combination of processes and messages](https://elixir-lang.org/getting-started/mix-otp/agent.html). What are typically long lived objects holding state in Python, in Elixir are long-running processes that you can communicate with messages.

I also wondered how polymorphism would work in a functional language, but Elixir solves it with [behaviours](https://elixir-lang.org/getting-started/typespecs-and-behaviours.html#behaviours) which define an interface for a _module_, specifying the functions that must be implemented. And [protocols](https://elixir-lang.org/getting-started/protocols.html) which allow the developer to define behaviour based on the data type.

Nowadays my Python code has become a lot more functional, I certainly avoid using classes and state more than I used to, although it doesn't feel quite right sometimes.

## Conclusion

I love Elixir and I love Python.

As with anything there are always trade-offs. I'm quite happy I made the [shoutouts.dev](https://shoutouts.dev) journey in Elixir, not to say it was easy but I've reached a point where I feel productive and am able to iterate quickly. The experience of learning and using Elixir has helped me appreciate some things about Python and to think about problems from a different, more functional, perspective.

I encourage you to have a look at what Elixir has to offer and in general to branch out and learn new languages.

And, of course, please check out [shoutouts.dev](https://shoutouts.dev), and let me know if you have any comments or ideas for it in its [repo](https://github.com/yeraydiazdiaz/shoutouts.dev).

Thanks for reading!

---

[^1]: [Elixir in Action](https://www.manning.com/books/elixir-in-action) by Saša Jurić, and [Programming Phoenix](https://pragprog.com/titles/phoenix14/programming-phoenix-1-4/) by Chris McCord, Bruce Tate and José Valim.
[^2]: I chose not to because I wanted to experiment with the [Erlang distributed capabilities](http://erlang.org/doc/reference_manual/distributed.html).
[^3]: This summary doesn't really do justice to concurrency aspect of Elixir/Erlang, Saša Jurić does a much better job at explaining all of this in his excellent talk [The Soul of Erlang and Elixir](https://www.youtube.com/watch?v=JvBT4XBdoUE).
</div>
