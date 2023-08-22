---
layout: post
title: "Automagic Python virtual environments with asdf and direnv"
date: 2023-08-22 12:19:15 +0000
categories: python
tags: python
short_description: Automagic Python virtual environments with asdf and direnv
keywords: "python,asdf,direnv"
---

<div markdown="1" class="sticky">
* TOC
{:toc}
</div>

<div markdown="1" id="text">

There are a LOT of ways to setup Python in your machine.

You may find it's already installed, or you can install it from [python.org](https://python.org),
or your OS's package manager,
or you can build it from source. But if you've been working with Python for a while, you're bound
to want more than one version of Python on your system. You may be working on a library and want support
for more than one version, or the version
used by your employer is not quite the latest and you want
to use all the new stuff in your side project.

There are also a LOT of ways to manage your virtual environments in Python. As you know if
you've worked in Python... well, at all, creating, activating, and deactivating virtual environments
is a big part of the workflow. So naturally there are many tools that address this.

This creates a combinatorial explosion of tools that results in basically everyone having their own
way of setting up Python for development. In this article I'd like to share _my_ combination
and why I like it.

**TL;DR** I use [`asdf`](https://asdf-vm.com/) to install different versions of Python combined
with [`direnv`](https://direnv.net/) via [`asdf-direnv`](https://github.com/asdf-community/asdf-direnv)
for virtual environment management. Sounds complicated? I promise I have valid reasons.

<!--more-->

## Installing all the Pythons

I work and mantain a few libraries that support many versions of Python, so in order to run the tests
I need a way to install each version separately and have all of them available for [tox](https://tox.wiki/en/latest/)
or [nox](https://nox.thea.codes/en/stable/) to run the test suite on each version.

This is something you simply cannot solve using your OS package manager since it's designed to install one specific version.
Naturally there are tools that tackle this problem and the most popular one is
[pyenv](https://github.com/pyenv/pyenv).

The idea, if you're not familiar with it, is to expose a command to install any version of Python
and allow the user to set up a global version and any number of _local_ versions that will become
active as the user enters a directory. This is done with simple commands like `pyenv install 3.10`,
`pyenv global 3.10.4`, and `pyenv local 3.9.12`, the latter will write to a special file in the
directory that tells pyenv to use that specific version of Python when entering it.

This principle is, of course, applicable and useful for all programming languages, `pyenv` is in fact
a fork of Ruby's `rbenv`, and there are many other tools for other similar programming languages like
Node.js's `nvm`. So, naturally, a single solution for all languages emerged:
[`asdf`](https://asdf-vm.com/).

`asdf` works the same way as `pyenv` and similar tools but allows a series of plugins for each
language. The commands are essentially the same but you only need to add the language, something
like `asdf local python 3.10.4`. The Python plugin for `asdf` actually uses `pyenv`'s
[`python-build`](https://github.com/pyenv/pyenv/blob/master/plugins/python-build/README.md)
to install the versions of Python, so all the helpful building hints described in
[`pyenv`'s wiki](https://github.com/pyenv/pyenv/wiki) still apply.

You may be wondering how do these tools know that if you type `python` in a specific
directory, one particular version of Python needs to be executed and not another. After all,
when you run any command in your shell the executable needs to be present in your `PATH`, and if
there are two of them surely the first one would be executed which may not be the one you want.

The solution: [**shims**](https://en.wikipedia.org/wiki/Shim_(computing)).

[`pyenv` has a great explanation for them](https://github.com/pyenv/pyenv#understanding-shims), but
essentially when you type `python` you don't actually run Python itself but small shell script
that resolves which version of Python you want and executes it.

Why am I mentioning this? You shall soon find out... 🧐

## The virtual environment management spectrum

Managing Python virtual environments is a very personal thing. Some people like no managing at all and
have developed muscle memory for creating and (de)activating the enviroments as they move between projects.

Others rely on higher level tools like [Poetry](https://python-poetry.org/) or [Hatch](https://hatch.pypa.io/latest/)
to manage the virtual environments along with other aspects of your project, like dependencies,
packaging or or publishing.

A separate issue is where you like your virtual environments to be located. Many people like having
the virtual environments in their project's directory, others prefer all virtual environments to
be all together somewhere else in the file system. Others don't care as long as it works.

Personally, I like to have them colocated with the project and activate as I enter the project's directory.

This type of workflow is exactly what the classic
[`virtualenvwrapper`](https://virtualenvwrapper.readthedocs.io/en/latest/) library does.
But you'll notice it's also what `pyenv` does for Python versions, so the good people behind
`pyenv` created [`pyenv-virtualenvwrapper`](https://github.com/pyenv/pyenv-virtualenvwrapper)
for those who are used to the `virtualenvwrapper` workflow, and
[`pyenv-virtualenv`](https://github.com/pyenv/pyenv-virtualenv) which works very similarly
but extends the `pyenv` command with virtual environment management capabilities.

"Ok, but what if you're using `asdf`?", you ask.

Well, since virtual environments are a very Python
thing it doesn't fit as neatly into its language agnostic API so there's no plugin to help you with it.
But all is not lost...

## Environment variables interlude

If you're a fan of the [12-factor app](https://12factor.net/codebase), or if you use environment
variables at all in your code, you may be familiar with [`direnv`](https://direnv.net/). But
in case you're not, `direnv` an excellent tool that allows you to define
enviroment variables in a `.envrc` file on your project's directory, and when you `cd` into it, it will
export them automatically. Once you `cd` out of it, `direnv` will `unset` them.

That in itself is amazingly useful, but it turns out you can do [much more](https://github.com/direnv/direnv/wiki),
including creating and activate virtual environments![^1]

This is exactly what I wanted: I can use `asdf` to set a local Python version for my project, and have
`direnv` create and activate the virtual environment as I enter the directory. Perfect!

But alas, all was not well in the land...

## The shim issue

Remember when I mentioned how shims work before? Turns out those small executables can run a fair
bit of code and they run on each call of each shimmed command which can be noticable,
particularly if you're on a slow machine.

Additionally, if you're used to the `which` command to know location of an executable,
e.g. `which python`, you will always get the same answer: the location of the shim, which is _not_
the one you want[^2]. This is because `pyenv` et al do **not** change the `PATH` environment variable.

This may not sound like a big deal, and it wasn't, until it was. I had a very obscure error when trying
to run the integration test suite for [`lunr.py`](https://github.com/yeraydiazdiaz/lunr.py) using
[`tox`](https://tox.wiki/en/latest/) where the `node` executable was not correctly resolved by the
shim in the `tox` environment, but manually adding its location to `PATH` worked.

However, there is a solution!

## The solution

Turns out you can install `direnv` using `asdf` using
[`asdf-direnv`](https://github.com/asdf-community/asdf-direnv)! It will also intregrate them, so
that when you enter a directory, it will set the appropriate locations of the executables in your
`PATH` using `direnv`, removing the shimming step and addressing the issues I mentioned above.
[`asdf-direnv` goes into detail about the motivations](https://github.com/asdf-community/asdf-direnv#motivation-or-shims-de-motivation)
and describing the solution in more detail.

What does that look like specifically?

1. Clone a repo or create a new directory to work on
2. Set the local Python version, e.g. `asdf local python 3.10.4`
3. Create a `.envrc` file with the following:

	```
	use asdf
	layout python python3.10
	```

4. `direnv` will detect the file but require your specific approval via `direnv allow`

At this point `asdf-direnv` will resolve and set the appropriate executables in PATH and `direnv`
will create any virtual environments in `.direnv` and activate them.

As I exit the directory the virtual environment will deactivate and the PATH will be restored.
When I enter the directory the virtual environment will activate and PATH will be populated along
with any environment variables I have declared in `.envrc`.

There is one catch though: because the PATH is changed when entering the directory, if you install
a new library that exposes a new executable it will not be immediately available. You must either
exit and enter the directory, or run `direnv reload`.

## Wrapping it all up

So we have `asdf` to install Python versions, `direnv` which takes care of the virtual
enviroments per project *and* enviroment variables, and `asdf-direnv` connecting the two.
It's blazing fast on any machine and `which` works as expected. If something goes wrong or you want to
start over, simply delete `.direnv` and run `direnv reload`.

I'm sure at this point you're wincing, thinking how you prefer your own setup. That's fine, this one
ticks all the boxes for me and my projects.


---

[^1]: Hat tip to the great Hynek Schlawack [who first pointed it out](https://hynek.me/til/python-project-local-venvs/).
[^2]: Which is why `pyenv` actually includes its own version `pyenv which <COMMAND>`.
