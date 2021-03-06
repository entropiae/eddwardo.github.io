---
layout: post
title: Linking Elixir processes together
date: 2015-11-04
summary: You are familiar with basic concept of Elixir/Erlang processes and you want to take a step further.
categories: elixir links
---

## Intro

If you actually don't know much about Elixir processes but you already learned something about Elixir checkout [my post about processes]({% post_url 2015-10-22-elixir-pingpong-table %}) and come back afterwords. I assume that you know how **spawn/1** and **spawn/3** work and you know basic concepts around process communication.


## Connecting processes together.

It is very usual that processes are somehow related to each other, we also know that they do not share state. Right now you should be asking: **"How to make sure that life of process depends on another?"**. Erlang provides mechanism called linking. In Elixir we can used it by calling **spawn_link**. Lets check what docs have to say about that function.

{% highlight elixir %}
def spawn_link(fun)

Spawns the given function, links it to the current process and returns its pid.

Check the Process and Node modules for other functions to handle processes,
including spawning functions in nodes.

Inlined by the compiler.

Examples

┃ current = self()
┃ child   = spawn_link(fn -> send current, {self(), 1 + 2} end)
┃
┃ receive do
┃   {^child, 3} -> IO.puts "Received 3 back"
┃ end
{% endhighlight %}

In my opinion that description is quite vage, let's test on our own then.

{% highlight elixir %}
iex(4)> child   = spawn_link(fn -> send current, {self(), 1 + 2} end)
#PID<0.65.0>
iex(5)> receive do
...(5)>   {^child, 3} -> IO.puts "Received 3 back"
...(5)> end
Received 3 back
:ok
iex(6)>
{% endhighlight %}

We received 3 which is OK, but

> Q: what if we call spawn instead of spawn_link?

{% highlight elixir %}
iex(1)> current = self()
#PID<0.57.0>
iex(2)> child   = spawn(fn -> send current, {self(), 1 + 2} end)
#PID<0.60.0>
iex(3)> receive do
...(3)>   {^child, 3} -> IO.puts "Received 3 back"
...(3)> end
Received 3 back
:ok
{% endhighlight %}

> Q: The result is exactly the same, so what is the exact purpose of **spawn_link** then?

> A: The answer is: "Error handling"

Or to be more specific: "spawn_link will notify linked process about abnormal exit reason of the dependent process". But before going any deeper we need to really understand what actually happens when process finishes its work.

## When process ends

<span style="float: right; padding-left: 20px;" markdown="1">
![oracle matrix meme]({{ site.baseurl }}/images/meme_oracle_happens_for_a_reason.jpg)
</span>
When process finishes its work, it exists. It is a different mechanism than exceptions and with it we can detect when something wrong (or unexpected) happened. When process finishes its work it implicitly calls **exit(:normal)** to communicate with its parent that job has been done. Every other argument to exit/1 than **:normal** will be treaded as an error. You should also know that Elixir shell is a process as well, so you should be able to link to it as well.

{% highlight elixir %}
iex(1)> spawn_link(fn -> exit(:normal) end)
#PID<0.104.0>
iex(2)> spawn(fn -> exit(:normal) end)
#PID<0.106.0>
{% endhighlight %}

By far there are no changes between **spawn/1** and **spawn_link/1**, that is because we exit the process with **:normal** reason. But what would happen for other reasons?
{% highlight elixir %}
{iex(1)> spawn(fn -> exit(:oh_no_i_did_something_wrong) end)
#PID<0.114.0>
iex(2)> spawn_link(fn -> exit(:oh_no_i_did_something_wrong) end)
** (EXIT from #PID<0.112.0>) :oh_no_i_did_something_wrong

Interactive Elixir (1.1.1) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)>
{% endhighlight %}

BINGO! Our shell processes did not catch **:EXIT** message, so link propagated the exit reason further to the process that observers elixir. The "observer" caught the error and restarted the shell process. But how to actually handle exit messages from linked processes?

## Trapping exists

Now that we know to track exits from linked processes, the question is:

> Q: How to actually react on failures of linked processes?

> A: trap_exit

Each process can be flagged, meaning you can customize its properties like minimal heap size, priority level, **trapping** and many more advanced things. [Erlang's documentation lists them all](http://www.erlang.org/doc/man/erlang.html#process_flag-2). The most interesting part is this:

> Setting trapping flag to true means that, exit signals arriving to a process are converted to **{'EXIT', from, reason}** messages, which can be received as ordinary messages. If trap_exit is set to false, the process exits if it receives an exit signal other than normal and the exit signal is propagated to its linked processes. Application processes should normally not trap exits.

So setting trapping flag is not a usual thing, it is because OTP from Erlang provides special building blocks for managing failures of other processes - **supervisors**. I will describe them in the next blog post, but for the time being we will go against the wind which is what curious programmers like most. Lets start with trapping exists from linked processes by setting the proper flag.


{% highlight elixir %}
iex(2)> Process.flag(:trap_exit, true)
false
{% endhighlight %}

As it says in the description, returned value is **false** which is a previous state of that flag. If you called it again it would return true. Now that we have trapping enabled lets link to the process which calculates 1 + 1 and finishes its work. We will call flush afterwords to receive incoming messages to the shell process.

{% highlight elixir %}
iex(3)> spawn_link(fn -> 1 + 1 end)
#PID<0.145.0>
iex(4)> flush
{:EXIT, #PID<0.145.0>, :normal}
:ok
{% endhighlight %}

Spawned process sent us **{:EXIT, #PID<0.145.0>, :normal}**. This means that process finished its work without any problems. Now, lets replicate that behavior more explicitly.

{% highlight elixir %}
iex(5)> spawn_link(fn -> exit(:normal) end)
#PID<0.164.0>
iex(6)> flush
{:EXIT, #PID<0.164.0>, :normal}
:ok
{% endhighlight %}

Now lets try to exit process with a message different that normal.

{% highlight elixir %}
iex(7)> spawn_link(fn -> exit(:custom_reason) end)
#PID<0.167.0>
iex(8)> flush
{:EXIT, #PID<0.167.0>, :custom_reason}
:ok
{% endhighlight %}

Great, we intercepted exit call and statement like

{% highlight elixir %}
receive do
  {:EXIT, pid, reason} -> here we could react
end
{% endhighlight %}

Would give us a way to react on processes exits. But what about exceptions? They cause processes to die too!

{% highlight elixir %}
iex(11)> spawn_link(fn -> raise "uuuuups!" end)
#PID<0.174.0>

22:28:14.246 [error] Process #PID<0.174.0> raised an exception
** (RuntimeError) uuuuups!
    :erlang.apply/2
iex(12)> flush
{:EXIT, #PID<0.174.0>,
 {\%RuntimeError{message: "uuuuups!"}, [{:erlang, :apply, 2, []}]}}
:ok
{% endhighlight %}

Great, exceptions are trapped as well, we could react on them as on processes exists.


## Visualizing links

Now that we have some knowledge about links, i'll show a visualization of simple demo. LinksTest module creates a chain of linked process and then tracks theirs exits.

Here is the code:
{% highlight elixir %}
defmodule LinksTest do
  def chain 0 do
    IO.puts "chain called with 0, wating 2000 ms before exit"
    :timer.sleep(2000)
    exit(:chain_breaks_here)
  end

  def chain n do
    Process.flag(:trap_exit, true)
    IO.puts "chain called with #{n}"
    :timer.sleep(1000)
    spawn_link(__MODULE__, :chain, [n-1])
    receive do
      {:EXIT, pid, reason} ->
        :timer.sleep(500)
        IO.puts "Child process exits with reason #{reason}"
    end
  end
end
{% endhighlight %}

And here is the demo

![chain demo]({{ site.base_url }}/images/process_chain.gif)

The interesting fact is, after exit with **:chain\_breaks_here**, next exists are :normal. It is because first process that catches :chain\_breaks_here exit code consumes it and then exits normally so error is swallowed by the first process that catches it. If we didn't trap exits in the chain and exit in the last process normally (:normal), the process would not exit at all - links prevent this. In other words: **When calling exit(:normal) the process will not finish, it will stay up**. Lets demo that as well!

Here is the code:
{% highlight elixir %}
defmodule LinksTestNoTrap do
  def chain 0 do
    IO.puts "normal exit in the last link"
    exit(:normal)
  end

  def chain n do
    IO.puts "create link in a chain no. #{n}"
    spawn_link(__MODULE__, :chain, [n-1])
    receive do
      msg -> IO.puts "#{n} received #{msg}"
    end
  end
end
{% endhighlight %}

And here is the demo

![chain demo]({{ site.base_url }}/images/process_chain_no_break.gif)


## Links are bidirectional

<span style="float: right; padding-left: 20px;" markdown="1">
![oracle matrix meme]({{ site.baseurl }}/images/3_musketeers_links.jpg)
</span>

By far we were killing processes that were children of other processes. The question is:

> Q: What if I kill the parent, will links break then?

> A: Yes, Links are bidirectional

Lets create a tree of processes and kill the root of that tree:

{% highlight elixir %}
defmodule Bidirectional do
  def leaf name do
    receive do
      msg -> IO.puts "#{name} received #{msg}"
    end
  end

  def node name do
    spawn_link __MODULE__, :leaf, ["node: #{name} first leaf"]
    spawn_link __MODULE__, :leaf, ["node: #{name} second leaf"]
    spawn_link __MODULE__, :leaf, ["node: #{name} third leaf"]
    receive do
      msg -> IO.puts "#{name} received #{msg}"
    end
  end

  def kernel do
    spawn_link __MODULE__, :node, ["first node"]
    spawn_link __MODULE__, :node, ["second node"]
    receive do
      msg -> IO.puts "kernel received #{msg}"
    end
  end

  def create_graph do
    spawn_link __MODULE__, :kernel, []
  end
end
{% endhighlight %}

The code is fairly simple, create_graph function creates a kernel process, which will spawn and link 2 node processes and each node will create 3 leafs. Then we will kill the kernel pid and see what is going to happen. Hard to image? Lets visualize that!


![bidirectional links demo]({{ site.base_url }}/images/links_on_a_tree.gif)

Everything happened as planned the entire tree has been killed. Death of a kernel processes caused a chain reaction and Everything died. Even our shell which was linked with a kernel died as well!
