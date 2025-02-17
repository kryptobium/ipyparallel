(parallel-task)=

# The IPython task interface

The task interface to the cluster presents the engines as a fault tolerant,
dynamic load-balanced system of workers. Unlike the direct interface, in
the task interface the user has no direct access to individual engines. By
allowing the IPython scheduler to assign work, this interface is simultaneously
simpler and more powerful.

Best of all, the user can use both of these interfaces running at the same time
to take advantage of their respective strengths. When the user can break up
the user's work into segments that do not depend on previous execution, the
task interface is ideal. But it also has more power and flexibility, allowing
the user to guide the distribution of jobs, without having to assign tasks to
engines explicitly.

## Starting the IPython controller and engines

To follow along with this tutorial, you will need to start the IPython
controller and four IPython engines. The simplest way of doing this is to use
the {command}`ipcluster` command:

```
$ ipcluster start -n 4
```

For more detailed information about starting the controller and engines, see
our {ref}`introduction <parallel-overview>` to using IPython for parallel computing.

## Creating a `LoadBalancedView` instance

The first step is to import the IPython {mod}`ipyparallel`
module and then create a {class}`.Client` instance, and we will also be using
a {class}`LoadBalancedView`, here called `lview`:

```ipython
In [1]: import ipyparallel as ipp

In [2]: rc = ipp.Client()
```

This form assumes that the controller was started on localhost with default
configuration. If not, the location of the controller must be given as an
argument to the constructor:

```ipython
# for a visible LAN controller listening on an external port:
In [2]: rc = ipp.Client('tcp://192.168.1.16:10101')
# or to connect with a specific profile you have set up:
In [3]: rc = ipp.Client(profile='mpi')
```

For load-balanced execution, we will make use of a {class}`LoadBalancedView` object, which can
be constructed via the client's {meth}`load_balanced_view` method:

```ipython
In [4]: lview = rc.load_balanced_view() # default load-balanced view
```

```{seealso}
For more information, see the in-depth explanation of {ref}`Views <parallel-details>`.
```

## Quick and easy parallelism

In many cases, you want to apply a Python function to a sequence of
objects, but _in parallel_. Like the direct interface, these can be
implemented via the task interface. The exact same tools can perform these
actions in load-balanced ways as well as multiplexed ways: a parallel version
of {func}`map` and {func}`@view.parallel` function decorator. If one specifies the
argument `balanced=True`, then they are dynamically load balanced. Thus, if the
execution time per item varies significantly, you should use the versions in
the task interface.

### Parallel map

To load-balance {meth}`map`, use a LoadBalancedView:

```ipython
In [62]: lview.block = True

In [63]: serial_result = map(lambda x:x**10, range(32))

In [64]: parallel_result = lview.map(lambda x:x**10, range(32))

In [65]: serial_result==parallel_result
Out[65]: True
```

### Parallel function decorator

Parallel functions are just like normal functions, but they can be called on
sequences and _in parallel_. The direct interface provides a decorator
that turns any Python function into a parallel function:

```ipython
In [10]: @lview.parallel()
   ....: def f(x):
   ....:     return 10.0*x**4
   ....:

In [11]: f.map(range(32))    # this is done in parallel
Out[11]: [0.0,10.0,160.0,...]
```

(parallel-dependencies)=

## Dependencies

Often, pure atomic load-balancing is too primitive for your work. In these cases, you
may want to associate some kind of `Dependency` that describes when, where, or whether
a task can be run. In IPython, we provide two types of dependencies:
[Functional Dependencies] and [Graph Dependencies]

```{note}
It is important to note that the pure ZeroMQ scheduler does not support dependencies,
and you will see errors or warnings if you try to use dependencies with the pure
scheduler.
```

### Functional Dependencies

Functional dependencies are used to determine whether a given engine is capable of running
a particular task. This is implemented via a special {class}`Exception` class,
{class}`UnmetDependency`, found in `ipyparallel.error`. Its use is very simple:
if a task fails with an UnmetDependency exception, then the scheduler, instead of relaying
the error up to the client like any other error, catches the error, and submits the task
to a different engine. This will repeat indefinitely, and a task will never be submitted
to a given engine a second time.

You can manually raise the {class}`UnmetDependency` yourself, but IPython has provided
some decorators for facilitating this behavior.

There are two decorators and a class used for functional dependencies:

```ipython
In [9]: import ipyparallel as ipp
```

#### @ipp.require

The simplest sort of dependency is requiring that a Python module is available. The
`@ipp.require` decorator lets you define a function that will only run on engines where names
you specify are importable:

```ipython
In [10]: @ipp.require('numpy', 'zmq')
   ....: def myfunc():
   ....:     return dostuff()
```

Now, any time you apply {func}`myfunc`, the task will only run on a machine that has
numpy and pyzmq available, and when {func}`myfunc` is called, numpy and zmq will be imported.
You can also require specific objects, not just module names:

```python
def foo(a):
    return a*a

@ipp.require(foo)
def bar(b):
    return foo(b)

@ipp.require(bar)
def baz(c, d):
    return bar(c) - bar(d)

view.apply_sync(baz, 4, 5)
```

#### @ipp.depend

The `@ipp.depend` decorator lets you decorate any function with any _other_ function to
evaluate the dependency. The dependency function will be called at the start of the task,
and if it returns `False`, then the dependency will be considered unmet, and the task
will be assigned to another engine. If the dependency returns _anything other than
\`\`False\`\`_, the rest of the task will continue.

```ipython
In [10]: def platform_specific(plat):
   ....:    import sys
   ....:    return sys.platform == plat

In [11]: @ipp.depend(platform_specific, 'darwin')
   ....: def mactask():
   ....:    do_mac_stuff()

In [12]: @ipp.depend(platform_specific, 'nt')
   ....: def wintask():
   ....:    do_windows_stuff()
```

In this case, any time you apply `mactask`, it will only run on an OSX machine.
`@ipp.depend` is like `apply`, in that it has a `@ipp.depend(f,*args,**kwargs)`
signature.

#### dependents

You don't have to use the decorators on your tasks, if for instance you may want
to run tasks with a single function but varying dependencies, you can directly construct
the {class}`dependent` object that the decorators use:

% sourcecode::ipython
%
% In [13]: def mytask(*args):
% ....: dostuff()
%
% In [14]: mactask = dependent(mytask, platform_specific, 'darwin')
% # this is the same as decorating the declaration of mytask with @ipp.depend
% # but you can do it again:
%
% In [15]: wintask = dependent(mytask, platform_specific, 'nt')
%
% # in general:
% In [16]: t = dependent(f, g, *dargs, **dkwargs)
%
% # is equivalent to:
% In [17]: @ipp.depend(g, \*dargs, **dkwargs)
% ....: def t(a,b,c):
% ....: # contents of f

### Graph Dependencies

Sometimes you want to restrict the time and/or location to run a given task as a function
of the time and/or location of other tasks. This is implemented via a subclass of
{class}`set`, called a {class}`Dependency`. A Dependency is a set of `msg_ids`
corresponding to tasks, and a few attributes to guide how to decide when the Dependency
has been met.

The switches we provide for interpreting whether a given dependency set has been met:

any|all

: Whether the dependency is considered met if _any_ of the dependencies are done, or
only after _all_ of them have finished. This is set by a Dependency's {attr}`all`
boolean attribute, which defaults to `True`.

success \[default: True\]

: Whether to consider tasks that succeeded as fulfilling dependencies.

failure \[default

: Whether to consider tasks that failed as fulfilling dependencies.
using `failure=True,success=False` is useful for setting up cleanup tasks, to be run
only when tasks have failed.

Sometimes you want to run a task after another, but only if that task succeeded. In this case,
`success` should be `True` and `failure` should be `False`. However sometimes you may
not care whether the task succeeds, and always want the second task to run, in which case you
should use `success=failure=True`. The default behavior is to only use successes.

There are other switches for interpretation that are made at the _task_ level. These are
specified via keyword arguments to the client's {meth}`apply` method.

after,follow

: You may want to run a task _after_ a given set of dependencies have been run and/or
run it _where_ another set of dependencies are met. To support this, every task has an
`after` dependency to restrict time, and a `follow` dependency to restrict
destination.

timeout

: You may also want to set a time-limit for how long the scheduler should wait before a
task's dependencies are met. This is done via a `timeout`, which defaults to 0, which
indicates that the task should never timeout. If the timeout is reached, and the
scheduler still hasn't been able to assign the task to an engine, the task will fail
with a {class}`DependencyTimeout`.

```{note}
Dependencies only work within the task scheduler. You cannot instruct a load-balanced
task to run after a job submitted via the MUX interface.
```

The simplest form of Dependencies is with `all=True, success=True, failure=False`. In these cases,
you can skip using Dependency objects, and pass msg_ids or AsyncResult objects as the
`follow` and `after` keywords to {meth}`client.apply`:

```ipython
In [14]: client.block=False

In [15]: ar = lview.apply(f, args, kwargs)

In [16]: ar2 = lview.apply(f2)

In [17]: with lview.temp_flags(after=[ar,ar2]):
   ....:    ar3 = lview.apply(f3)

In [18]: with lview.temp_flags(follow=[ar], timeout=2.5)
   ....:    ar4 = lview.apply(f3)
```

```{seealso}
Some parallel workloads can be described as a [Directed Acyclic Graph](https://en.wikipedia.org/wiki/Directed_acyclic_graph), or DAG. See {ref}`DAG Dependencies <dag-dependencies>` for an example demonstrating how to use map a NetworkX DAG
onto task dependencies.
```

#### Impossible Dependencies

The schedulers do perform some analysis on graph dependencies to determine whether they
are not possible to be met. If the scheduler does discover that a dependency cannot be
met, then the task will fail with an {class}`ImpossibleDependency` error. This way, if the
scheduler realized that a task can never be run, it won't sit indefinitely in the
scheduler clogging the pipeline.

The basic cases that are checked:

- depending on nonexistent messages
- `follow` dependencies were run on more than one machine and `all=True`
- any dependencies failed and `all=True,success=True,failures=False`
- all dependencies failed and `all=False,success=True,failure=False`

```{warning}
This analysis has not been proven to be rigorous, so it is likely possible for tasks
to become impossible to run in obscure situations, so a timeout may be a good choice.
```

## Retries and Resubmit

### Retries

Another flag for tasks is `retries`. This is an integer, specifying how many times
a task should be resubmitted after failure. This is useful for tasks that should still run
if their engine was shutdown, or may have some statistical chance of failing. The default
is to not retry tasks.

### Resubmit

Sometimes you may want to re-run a task. This could be because it failed for some reason, and
you have fixed the error, or because you want to restore the cluster to an interrupted state.
For this, the {class}`Client` has a {meth}`rc.resubmit` method. This takes one or more
msg_ids, and returns an {class}`AsyncHubResult` for the result(s). You cannot resubmit
a task that is pending - only those that have finished, either successful or unsuccessful.

(parallel-schedulers)=

## Schedulers

There are a variety of valid ways to determine where jobs should be assigned in a
load-balancing situation. In IPython, we support several standard schemes, and
even make it easy to define your own. The scheme can be selected via the `scheme`
argument to {command}`ipcontroller`, or in the {attr}`TaskScheduler.schemename` attribute
of a controller config object.

The built-in routing schemes:

To select one of these schemes:

```
$ ipcontroller --scheme=<schemename>
for instance:
$ ipcontroller --scheme=lru
```

lru: Least Recently Used

> Always assign work to the least-recently-used engine. A close relative of
> round-robin, it will be fair with respect to the number of tasks, agnostic
> with respect to runtime of each task.

plainrandom: Plain Random

> Randomly picks an engine on which to run.

twobin: Two-Bin Random

> **Requires numpy**
>
> Pick two engines at random, and use the LRU of the two. This is known to be better
> than plain random in many cases, but requires a small amount of computation.

leastload: Least Load

> **This is the default scheme**
>
> Always assign tasks to the engine with the fewest outstanding tasks (LRU breaks tie).

weighted: Weighted Two-Bin Random

> **Requires numpy**
>
> Pick two engines at random using the number of outstanding tasks as inverse weights,
> and use the one with the lower load.

### Greedy Assignment

Tasks can be assigned greedily as they are submitted. If their dependencies are
met, they will be assigned to an engine right away, and multiple tasks can be
assigned to an engine at a given time. This limit is set with the
`TaskScheduler.hwm` (high water mark) configurable in your
{file}`ipcontroller_config.py` config file, with:

```python
# the most common choices are:
c.TaskSheduler.hwm = 0 # (minimal latency, default in IPython < 0.13)
# or
c.TaskScheduler.hwm = 1 # (most-informed balancing, default in ≥ 0.13)
```

In IPython \< 0.13, the default is 0, or no-limit. That is, there is no limit to the number of
tasks that can be outstanding on a given engine. This greatly benefits the
latency of execution, because network traffic can be hidden behind computation.
However, this means that workload is assigned without knowledge of how long
each task might take, and can result in poor load-balancing, particularly for
submitting a collection of heterogeneous tasks all at once. You can limit this
effect by setting hwm to a positive integer, 1 being maximum load-balancing (a
task will never be waiting if there is an idle engine), and any larger number
being a compromise between load-balancing and latency-hiding.

In practice, some users have been confused by having this optimization on by
default, so the default value has been changed to 1 in IPython 0.13. This can be slower,
but has more obvious behavior and won't result in assigning too many tasks to
some engines in heterogeneous cases.

### Pure ZMQ Scheduler

For maximum throughput, the 'pure' scheme is not Python at all, but a C-level
{class}`MonitoredQueue` from PyZMQ, which uses a ZeroMQ `DEALER` socket to perform all
load-balancing. This scheduler does not support any of the advanced features of the Python
{class}`.Scheduler`.

Disabled features when using the ZMQ Scheduler:

- Engine unregistration
  : Task farming will be disabled if an engine unregisters.
  Further, if an engine is unregistered during computation, the scheduler may not recover.
- Dependencies
  : Since there is no Python logic inside the Scheduler, routing decisions cannot be made
  based on message content.
- Early destination notification
  : The Python schedulers know which engine gets which task, and notify the Hub. This
  allows graceful handling of Engines coming and going. There is no way to know
  where ZeroMQ messages have gone, so there is no way to know what tasks are on which
  engine until they _finish_. This makes recovery from engine shutdown very difficult.

```{note}
TODO: performance comparisons
```

## More details

The {class}`LoadBalancedView` has many more powerful features that allow quite a bit
of flexibility in how tasks are defined and run. The next places to look are
in the following classes:

- {class}`~ipyparallel.client.view.LoadBalancedView`
- {class}`~ipyparallel.client.asyncresult.AsyncResult`
- {meth}`~ipyparallel.client.view.LoadBalancedView.apply`
- {mod}`~ipyparallel.controller.dependency`

The following is an overview of how to use these classes together:

1. Create a {class}`Client` and {class}`LoadBalancedView`
2. Define some functions to be run as tasks
3. Submit your tasks to using the {meth}`apply` method of your
   {class}`LoadBalancedView` instance.
4. Use {meth}`.Client.get_result` to get the results of the
   tasks, or use the {meth}`AsyncResult.get` method of the results to wait
   for and then receive the results.

```{seealso}
A demo of {ref}`DAG Dependencies <dag-dependencies>` with NetworkX and IPython.
```
