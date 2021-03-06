# Within batch parallelism

## Content
1. [Basic usage](#basic-usage)
1. [Parallelized methods](#parallelized-methods)
1. [Decorator arguments](#decorator-arguments)
1. [Additional decorator arguments](#additional-decorator-arguments)
1. [Init function](#init-function)
1. [Post function](#post-function)
1. [Targets](#targets)
1. [Arguments with default values](#arguments-with-default-values)
1. [Number of parallel jobs](#number-of-parallel-jobs)


## Basic usage
The `inbatch_parallel` decorator allows to run a method in parallel:

```python
from dataset import Batch, inbatch_parallel, action

class MyBatch(Batch):
    ...
    @action
    @inbatch_parallel(init='_init_fn', post='_post_fn', target='threads')
    def some_action(self, item, arg1, arg2):
        # process just one item
        return some_value
```

## Parallelized methods
You can parallelize actions as well as ordinary methods.

## Decorator arguments

### init='some_method'
Required.
The only required argument which contains a method name to be called to initialize the parallel execution.

### post='other_method'
Optional.
A method name which is called after all parallelized tasks are finished.

### target='threads'
Optional.
Specifies a parallelization engine, should be one of `threads`, `nogil`, `async`, `mpc`.


## Additional decorator arguments
You can pass any other arguments to the decorator and they will be passed further to `init` and `post` functions.

```python
class MyBatch(Batch):
...
    @inbatch_parallel(init='_init_default', post='_post_default', target='threads', clear_data=False)
    def some_method(self, item):
        # process just one item
        return some_value

    def _init_default(self, clear_data):
        ...

    def _post_default(self, list_of_res, clear_data):
        ...
```

All these arguments should be named argments only. So you should not write like this:
```python
@inbatch_parallel('_init_default', '_post_default', 'threads', clear_data)
```
It might sometimes works though. But no guarantees.

The preferred way is:
```python
@inbatch_parallel(init='_init_default', post='_post_default', target='threads', clear_data=False)
```

Using this technique you can pass an action name to the `init` function:
```python
class MyBatch(Batch):
...
    @inbatch_parallel(init='_init_default', post='_post_default', target='threads', method='one_method')
    def one_method(self, item):
        # process just one item
        return some_value

    @inbatch_parallel(init='_init_default', post='_post_default', target='threads', method='some_other_method')
    def some_other_method(self, item):
        # process just one item
        return some_value
```
However, usually you might consider writing specific init / post functions for different actions.


## Init function
Init function defines how to parallelize the decorated method. It returns a list of arguments for each invocation of the parallelized action.
So if you want to run 10 parallel copies of the method, `init` should return a list of 10 items. Usually you run the method once for each item in the batch. However you might also run one method per 10 or 100 or any other number of items if it is beneficial for your specific circumstances (memory, performance, etc.)

The simplest `init` just returns a sequence of indices:
```python
class MyBatch(Batch):
...
    @action
    @inbatch_parallel(init='indices')
    def some_action(self, item_id)
        # process an item and return a value for that item
        return proc_value
```
For a batch of 10 items `some_action` will be called 10 times as `some_action(index1)`, `some_action(index2)`, ..., `some_action(index10)`.

You may define as many arguments as you need:
```python
class MyBatch(Batch):
...
    def _init_fn(self, *args, **kwargs):
        return [[self._data, item, another_arg, one_more_arg] for item in self.indices]
```
Here the action will be fired as:  
`some_action(self._data, index1, another_arg, one_more_arg)`  
`some_action(self._data, index2, another_arg, one_more_arg)`  
`...`

`item_args` does not have to be strictly a list, but any sequence - tuple, numpy array, etc - that supports the unpacking operation ([`*seq`](https://docs.python.org/3/tutorial/controlflow.html#unpacking-argument-lists)):

**Attention!** It cannot be a tuple of 2 arguments (see below why).

You can also pass named arguments:
```python
class MyBatch(Batch):
...
    def _init_fn(self, *args, **kwargs):
        return [dict(data=self._data, item=item, arg1=another_arg, arg2=one_more_arg) for item in self.indices]
```
And the action will be fired as:  
`some_action(data=self._data, item=index1, arg1=another_arg, arg2=one_more_arg)`  
`some_action(data=self._data, item=index2, arg1=another_arg, arg2=one_more_arg)`  
`...`

And you can also combine positional and named arguments:
```python
class MyBatch(Batch):
...
    def _init_fn(self, *args, **kwargs):
        return [tuple(list(self._data, item), dict(arg1=another_arg, arg2=one_more_arg)) for item in self.indices]
```
So the action will be fired as:  
`some_action(self._data, index1, arg1=another_arg, arg2=one_more_arg)`  
`some_action(self._data, index2, arg1=another_arg, arg2=one_more_arg)`  
`...`

Thus, 2-items tuple is reserved for this situation (1st item is a list of positional arguments and 2nd is a dict of named arguments).

That is why you cannot pass a tuple of 2 arguments:
```python
    ...
    item_args = tuple(some_arg, some_other_arg)
    ...
```
Instead make it a list:
```python
    ...
    item_args = list(some_arg, some_other_arg)
    ...
```

### Init's additional arguments
Take into account that all arguments passed into actions are also passed into the `init` function. So when you call:
```python
some_pipeline.some_parallel_action(10, 12, my_arg=12)
```
The `init` function will be called like that:
```python
init_function(10, 12, my_arg=12)
```
This is convenient when you need to initialize some additional variables depending on the arguments. For instance, to create a numpy array of a certain shape filled with specific values or set up a random state or even pass additional arguments back to action methods.

If you have specified [additional decorator arguments](#additional-decorator-arguments), they are also passed to the `init` function:
```python
init_function(10, 12, my_arg=12, arg_from_parallel_decorator=True)
```

### `run_once` for no parallelism
You cannot call an `async` action in pipelines, because `async`-methods should be `awaited` for. This is where `@inbatch_parallel` might be helpful without any parallelism whatsoever. All you need is `run_once` init-function:
```python
class MyBatch(Batch):
    ...
    @inbatch_parallel(init='run_once', target='async')
    async def read_some_data(self, src, fmt='raw'):
        ...
...
some_pipeline
    .do_whatever_you_want()
    .read_some_data('/some/path')
    .do_something_else()
```

## Post function
When all parallelized tasks are finished, the `post` function is called.

The first argument it receives is the list of results from each parallel task.
```python
class MyBatch(Batch):
    ...
    def _init_default(self, *args, **kwargs):
        ...
        return all_args

    def _post_default(self, list_of_res, *args, **kwargs):
        ...
        return self

    @action
    @inbatch_parallel(init='_init_default', post='_post_default')
    def some_action(self, item_id)
        # process an item and return a value for that item
        return proc_value
```
Here `_post_default` will be called as
```python
_post_default([proc_value_from_1, proc_value_from_2, ..., proc_value_from_last])
```

If anything went wrong, than instead of `proc_value`, there would be an instance of some `Exception` caught in the parallel tasks.

This is where `any_action_failed` might come in handy:
```python
from dataset import Batch, action, inbatch_parallel, any_action_failed

class MyBatch(Batch):
    ...
    def _post_fn(self, list_of_res, *args, **kwargs):
        if any_action_failed(list_of_res):
            # something went wrong
        else:
            # process the results
        return self

    @action
    @inbatch_parallel(init='indices', post='_post_fn')
    def some_action(self, item_id)
        # process an item and return a value for that item
        return proc_value
```

`Post`-function should return an instance of a batch class (not necessarily the same). Most of the time it would be just `self`.

If an action-method changes data directly, you don't need a `post`-function.
```python
from dataset import Batch, action, inbatch_parallel, any_action_failed

class MyBatch(Batch):
    ...
    @action
    @inbatch_parallel(init='indices')
    def some_action(self, item_id)
        # process an item and return a value for that item
        self._data[item_id] = new_value
```
Don't forget about GIL. A python function with `target=threads` won't give any performance increase, though this might simplify your code.
However, `numba` or `cython` allow for a real multithreading.
```python
from dataset import Batch, action, inbatch_parallel, any_action_failed
from numba import njit

@njit(nogil=True)
def change_data(data, index):
    # data is a numpy array
    data[index] = new_value


class MyBatch(Batch):
    ...
    def _init_numba(self, *args, **kwargs):
        return [[self.data, i] for i in self.indices]

    @action
    @inbatch_parallel(init='_init_numba', target='nogil')
    def some_action(self, item_id)
        return change_data
```
Here all batch items will be updated simultaneously.

## Targets
There are four targets available: `threads`, `nogil`, `async`, `mpc`

### threads
A method will be parallelized with [concurrent.futures.ThreadPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html#threadpoolexecutor).
Take into account that due to [GIL](https://wiki.python.org/moin/GlobalInterpreterLock) only one python thread is executed in any given moment (pseudo-parallelism).
However, a function with intensive I/O processing or waiting for some synchronization might get a considerable performance increase.

This is a default engine which is used if `target` is not specified in the `inbatch_parallel` decorator.

### nogil
To get rid of GIL you might write a [cython](http://cython.org/) or [numba](http://numba.pydata.org/) function which can run in parallel.
And a decorated method should just return this nogil-function which will be further parallelized.

```python
from numba import njit
from dataset import Batch, action, inbatch_parallel

@njit(nogil=True)
def numba_fn(data, index, arg):
    # do something
    return new_data

class MyBatch(Batch):
    ...
    @action
    @inbatch_parallel(init='_init_default', post='_post_default', target='nogil')
    def some_action(self, arg)
        # do not process anything, just return nogil-function
        return numba_fn
```

### async
For I/O-intensive processing you might want to consider writing an [`async` method](https://docs.python.org/3/library/asyncio-task.html).
```python
class MyBatch(Batch):
    ...
    @action
    @inbatch_parallel(init='_init_default', post='_post_default', target='async')
    async def some_action(self, item, some_arg)
        # do something
        proc_value = await other_async_function(some_arg)
        return proc_value
```

### mpc
With `mpc` you might run calculations in separate processes thus removing GIL restrictions. For this [concurrent.futures.ProcessPoolExecutor](https://docs.python.org/3/library/concurrent.futures.html#processpoolexecutor) is used. Likewise `nogil` the decorated method should just return a function which will be executed in a separate process.

```python
from dataset import Batch, action, inbatch_parallel

def mpc_fn(data, index, arg):
    # do something
    return new_data

class MyBatch(Batch):
    ...
    @action
    @inbatch_parallel(init='_init_default', post='_post_default', target='mpc')
    def some_action(self, arg)
        # do not process anything, just return a function which will be run as a separate process
        return mpc_fn
```
Multiprocessing requires all code and data to be serialized (with [pickle](https://docs.python.org/3/library/pickle.html)) in order to be sent to another process. And many classes and methods are not so easy (or even impossible) to pickle. That is why to parallelize functions might be a better choice. Nevertheless, with all these thoughts in mind you should carefully consider your parallelized function and the arguments it receives.

Besides, you might want to implement a thorough logging mechanism as multiprocessing configurations are susceptible to hanging up. Without logging it would be quite hard to understand what happened and then debug your code.


## Arguments with default values
If you have a function with default arguments, you may call it without passing those arguments.
```python
class MyBatch(Batch):
    ...
    @action
    @inbatch_parallel(init='_init_default', post='_post_default', target='mpc')
    def some_action(self, arg1, arg2, arg3=3)
        ...

# arg3 takes the default value = 3
batch.some_action(1, 2)
```
However, when you call it this way, the default arguments are not available externally (in particular, in decorators).
This is the problem for `nogil`/`mpc` parallelism.

The best solutions would be not to use default values at all, but if you really need them, you should copy them into parallelized functions:
```python
def mpc_fn(item, arg1, arg2, arg3=10):
    # ...

class MyBatch(Batch):
    ...
    @action
    @inbatch_parallel(init='_init_default', post='_post_default', target='mpc')
    def some_action(self, arg1, arg2, arg3=10)
        return mpc_fn
```

You might also return a [partial](https://docs.python.org/3/library/functools.html#functools.partial) with these arguments:
```python
from functools import

def mpc_fn(item, arg1, arg2, arg3=10):
    # ...

class MyBatch(Batch):
    ...
    @action
    @inbatch_parallel(init='_init_default', post='_post_default', target='mpc')
    def some_action(self, arg1, arg2, arg3=10)
        return partial(mpc_fn, arg3=arg3)
```

## Number of parallel jobs
By default each action runs as many parallel tasks as the number of cores your computer/server has. That is why sometimes you might want to run fewer or more tasks. Then you can specify this number in each action call with `n_workers` option:
```python
some_pipeline.parallel_action(some_arg, n_workers=3)
```
Here `parallel_action` will have only 3 parallel tasks being executed simultaneously. Others will wait in the queue.

**Attention!** You cannot use `n_workers` with `target=async`.
