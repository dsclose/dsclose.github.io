---
layout: post
title: "Clearing stdin in Python without blocking program flow."
date:   2016-04-26
categories: linux python
---

It turns out that clearing stdin is made easy with Python's `select` module.
If you're just interested in the code, [here's the Gist][gist].  Otherwise,
let's work through it.


It began with a question...
---------------------------

A few days ago I saw a question on StackOverflow entitled [How to clear stdin
before getting new input?][question] It was
a worthy question. Many a script takes a few seconds to perform tasks before
asking for input and we've all accidentally typed something into a console
whilst a task is going on. We don't want accidental input being read and
influencing a scripts behaviour.

Excited by a good Python problem I eagerly submitted [an answer][answer]
demonstrating how the `select` module can help us out. Of course, a few minutes
later a comment appeared:

> But you did notice that this is a **C** question, not a python question?
> *[@Jens](http://stackoverflow.com/users/648658/jens)*

Oh...


What we want...
---------------

I'll be leaving the answer there as a note-to-self in humility. Not all is lost
though as this functionality could be useful in various contexts. Reading the
output of a process, for example, may block our test run if the expected output
isn't there. Something like this would be nice:

```python
assert(has_unread_input(my_subprocess.output))
assert('expected' in my_subprocess.output.read())
```

So let's be precise for a moment. What we really want are two functions:

1. `has_unread_input` should take a readable file-descriptor and immediately
   return `True` if there is data waiting to be read or `False` otherwise.
2. `discard_input` should also take a readable file-descriptor and should
   immediately read any pending data and return it in a bytes construct.
3. `refreshed_input` should simply be a wrapper for the built-in `input`
   (or `raw_input` in Python 2) that precedes the call by clearing stdin.

To make everyone's life easier, the input-streams for each should default to
`sys.stdin`. That way we could do something like this:

```python
if has_unread_input(my_stream):  # non-blocking read
    data = my_stream.read()
```

Or maybe:

```python
time_consuming_preparation()
accidental = discard_input()     # clear unwanted input

if nothing_important(accidental):
    repeat = input('Would you like to continue? ').lower()
    if 'yes'.starts_with(repeat):
        some_destructive_task()
```

If we're feeling confident, we can simplify this to:

```python
time_consuming_preparation()

repeat = refreshed_input('Would you like to continue? ').lower()
if 'yes'.starts_with(repeat):
    some_destructive_task()
```

For all of this, there's only one Python module we'll need...

The select module.
------------------

The `select` module is in the Python standard library and allows us to wait
for different I/O events across a collection of streams. As with many stream
based operations the [Python documentation]
(https://docs.python.org/3/library/select.html) mentions an important
cross-platform caveat:

> Note that **on *Windows*, it only works for sockets;** on other operating
systems, it also works for other file types (in particular, on *Unix*, it works
on pipes). It cannot be used on regular files to determine whether a file has
grown since it was last read.

Those of you looking to do this on Windows should investigate [this answer on
StackOverflow](http://stackoverflow.com/a/2521054/4540711) which makes use of
the `msvcrt` module.

The Python docs use a rather opaque echo-server as an example of how to use the
module. Doug Hellmann does a nice job of breaking the example down on [his
module-of-the-week article][pymotw]. All we need to check for I/O events,
however, is the module's synonymous function. It takes a minimum of three
arguments:

```python
import select

select.select(rlist, wlist, xlist)
```

Each argument should be a list of file descriptors (so `[sys.sdtin]` in our
case) and the function then waits until a specific I/O operation is available.
These I/O operations are **r**ead, **w**rite or some other e**x**ception on the
given file descriptors. It returns a Tuple of corresponding lists populated
with the file descriptors that are ready.

So, if there *is* input waiting in `sys.stdin` then the function would behave
like so:

    >>> import select
    >>> import sys
    >>>
    >>> select.select([sys.stdin], [], [])
    ([sys.stdin], [], [])
    >>>

By itself, this doesn't solve our problem because by default the function will
wait until an I/O operation is available. Importantly, however, `select.select`
has an optional `timeout` argument denoting how long it will wait for an I/O
event before giving up. We simply have to set the timeout to zero and we can
check for input without blocking the program flow.

Let's see an example where there is no input waiting in `sys.stdin`:

```python
>>> import select
>>> import sys
>>>
>>> timeout = 0
>>> select.select([sys.stdin], [], [], timeout)
([], [], [])
>>>
```

Knowing that we only want the first element of that tuple (the read-event file-
descriptors) we're ready to make a useful if statement:

```python
if sys.stdin in select.select([sys.stdin], [], [], 0)[0]:
    print('Input is waiting to be read.')
```

That's all we'll need of the `select` module. Let's crack on with our functions.

Defining our functions.
----------------------
After we understand our if statement, above, writing `has_read_input` is just
a quick refactor away:

```python
import select
import sys

def has_unread_input(stream=sys.stdin, timeout=0):
    '''Where stream is a readable file-descriptor, returns
    True if the given stream has input waiting to be read.
    Otherwise returns False.

    The default timeout of zero indicates that the given
    stream should be polled for input and the function
    returns immediately. Increasing the timeout will cause
    the function to wait that many seconds for the poll to
    succeed. Setting timeout to None will cause the function
    to block until it can return True.
    '''

    if stream.readable():
        return stream in select.select([stream], [], [], timeout)[0]
    else:
        return False
```

It would be nice to follow this up with a similar pattern for `discard_input`:

```python
if has_available_input(sys.stdin):
   return sys.stdin.read()
else:
   return bytes()
```

Unfortunately, `read()` will block until it meets `EOF` so instead we need a
bit of iteration:

```python
def discard_input(stream=sys.stdin):
    '''Where stream is a readable file-descriptor, reads
    all available data and returns any bytes read. If no
    data is available, then an empty set of bytes is
    returned.
    '''

    data = bytearray()

    while has_unread_input(stream):
        data.extend(stream.readline())

    return bytes(data)
```

Finally, we can write a wrapper for the `input` built-in (or `raw_input` in
Python 2):

```python
def refreshed_input(prompt=''):
    '''Clears stdin before writing the given prompt and
    waiting for input from stdin. Returns the given
    input as a string with the trailing-newline removed.
    '''

    discard_input(sys.stdin)
    try:
        return raw_input(prompt)
    except NameError:
        return input(prompt)
```

The `discard_input` function can be used as a non-blocking way to clear input
streams and should work in both Python 2 and Python 3.

[gist]: https://gist.github.com/dsclose/ce9ebc5b9c345cca36a1f93150c2d4a7
[question]: http://stackoverflow.com/q/36715002/4540711
[answer]: http://stackoverflow.com/a/36718598/4540711
[pymotw]: https://pymotw.com/2/select/
