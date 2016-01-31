---
layout: post
title: "Profile and Pry-Debugging Elixir"
date: '2016-02-01'
description:
categories: ['elixir']
newb: true
versions:
    - name: Elixir
      version: '1.2'
---

In my heart of hearts, I'm a 'printf' debugger, where my first solution to any
problem is just throwing godawful amounts of `puts` in Ruby, `print`s in Python,
`IO.inspect`s in Elixir,  `console.log` in Javascript, and whatever else I can
do to generate a wall of text saying "OK HERE NOW" or "IT WORKED! GOT TO STEP 3".

Somewhat surprisingly, this doesn't always work, and that's where things like
`pry` in Ruby come in. Elixir also has a `pry` command, which is pretty simple
to use.

## Pry-Debugging

{%highlight elixir %}
require IEx; IEx.pry
{% endhighlight %}

And that's it. As far as I'm aware IEx's pry doesn't support any of the extra
commands that Ruby's pry does (like `whereami`, `find-method`, etc...) but at
least you can get a peek of what all the local variables in a method are. Which
is super useful.

To get Elixir's pry working you have to run the script through IEx with using
the `-s` flag. For tests, this means instead of `mix test` you do
`iex -S mix test`. When the execution hits the `IEx.pry` call, IEx will prompt
asking

```
  Request to pry #PID<0.315.0> at test/pry_test.exs:8. Allow? [Yn]
```

Once running, you'll be in a shell where you can access the local variables and
inspect them or pass them around to different functions. Unfortunately, after a
minute you'll get this error:

```
     ** (ExUnit.TimeoutError) test timed out after 60000ms. You can change the timeout:

       1. per test by setting "@tag timeout: x"
       2. per case by setting "@moduletag timeout: x"
       3. globally via "ExUnit.start(timeout: x)" configuration
       4. or set it to infinity per run by calling "mix test --trace"
          (useful when using IEx.pry)

     Timeouts are given as integers in milliseconds.
```

Thankfully, this explains what to do: Just change the command to
`iex -S mix test --trace` and you'll avoid the timeout. Elixir's docs can really
be 💯 sometimes. When you're done using IEx's pry, enter `respawn` in the
console to continue where the program left off.

## Profiling

Elixir 1.1 added the <a href="http://elixir-lang.org/docs/v1.1/mix/Mix.Tasks.Profile.Fprof.html">Fprof</a>
mix task, which formats profiling code a bit nicer than the default Erlang
formatting. It's easy enough to use the wrapper method that the mix task uses to
profile whatever snippet of code you want:

{%highlight elixir %}
  Mix.Tasks.Profile.Fprof.profile(
    fn -> YourModule.method(a, b, c) end,
    [sort: "own", callers: true]
  )
{% endhighlight %}

The last param is the keyword list of arguments, which you are described in the
docs for the Fprof mix task. I would like this to provide some more options so
that it's more in line with `ruby-prof`, which can output in different formats
so that it can be used by other tools, so hopefully someone else works on that
before it becomes an itch that I have to scratch.

Anywho, I hope this helps you debug and optimize your stuff, weary Elixir
searcher
