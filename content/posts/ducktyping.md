---
title: "Ducktyping & Protocols"
date: 2020-01-23T19:03:13+01:00
tags: ["programming", "python"]
summary: "Writing protocols in Python."
---

When writing a code, there are often times,
where you need some function to operate on some object independetly without worrying about its type
or what class excatly it is - just accessing some attributes or calling some methods.
In dynamically typed languages, this is called **ducktyping**.

## Ducktyping

> "If it looks like a duck and quacks like a duck, it's a duck."

Since Python is dynamically typed, this is actually very easy to do:

```Python
"""Python3 ducktyping demo"""
from typing import Any

class Duck(object):
    def __init__(self, name):
        self.name = name

    def get_sound(self):
        return "Quack!"


def make_sound(animal):
    """Print the sound of the animal."""
    print(f"{animal.name} does '{animal.get_sound()}'")

if __name__ == "__main__":
    make_sound(Duck("Duck named Bob"))
```

The result when we run this script:

```bash
> python ducktyping.py
Duck named Bob does 'Quack!'
```

Cool! Excatly what we wanted.
In fact, this will work for any object with `.name` attribute and `.make_sound()` callable.
However, if we try to run this with some object other than the ones we're ready for,
exception is going to be raised:

```Python
"""Python3 ducktyping demo"""
from typing import Any

class Duck(object):
    def __init__(self, name):
        self.name = name

    def get_sound(self):
        return "Quack!"

class Stone(object):
    def __init__(self, name):
        self.name = name

def make_sound(animal):
    """Print the sound of the animal."""
    print(f"{animal.name} does '{animal.get_sound()}'")

if __name__ == "__main__":
    make_sound(Duck("Duck named Bob"))
    # Notice the new make_sound call with a Stone object,
    # which does not have a `get_sound` method:
    make_sound(Stone("ROCKy Balboa"))
```

Running this gives us:

```bash
> python ducktyping.py
Duck named Bob does 'Quack!'
Traceback (most recent call last):
  File "py3.py", line 21, in <module>
    make_sound(Stone("ROCKy Balboa"))
  File "py3.py", line 17, in make_sound
    print(f"{animal.name} does '{animal.make_sound()}'")
AttributeError: 'Stone' object has no attribute 'make_sound'
```

Ouch... The editor did not raise any warning, running the code was fine right until we encountered the second `make_sound()` call.
But hey, these are the struggles of dynamically typed, interpreted languages.
But what if we wanted to catch some of these errors before they actually happen?
Is there a way to prevent this? Of course there is! Here's where static type checkers come in!
Actually, there was one available all along, we just didn't use it ¯\\\_(ツ)_/¯.
Since Python3.5, there's a [mypy](https://github.com/python/mypy) bundled with Python distribution.

Let's try to add some type annotations to this function to make our code less error-prone.
First of, let's try to add type annotation for the first parameter and return value.
The type annotation itself will be just the `Duck` class:

```Python
def make_sound(animal: Duck) -> None:
```

This will be the new `make_sound` function declaration.
Since the `make_sound` function does not return anything,
we will say it returns `None`.
Let's try to run `mypy` from command line and provide the file with the code:

```Bash
> mypy ducktyping.py
ducks_py3.py:21: error: Argument 1 to "make_sound" has incompatible type "Stone"; expected "Duck"
```

Woah! `mypy` just told us that we're trying to call `make_sound` on a class `Stone`,
which is not compatible with `Duck`! Neat.
But don't be fooled, this does not fix our problem.
In fact, the code still works the same way:

```bash
> python ducktyping.py
Duck named Bob does 'Quack!'
Traceback (most recent call last):
  File "py3.py", line 21, in <module>
    make_sound(Stone("ROCKy Balboa"))
  File "py3.py", line 17, in make_sound
    print(f"{animal.name} does '{animal.make_sound()}'")
AttributeError: 'Stone' object has no attribute 'make_sound'
```

We still get the same error, but with `mypy`,
we're able to discover it before executing the code.

## Protocols

But what if we try to add some other class that implements the `name` attribute and `make_sound()` method?
Let's implement a `Dog` class:

```Python
class Dog(object):
    def __init__(self, name):
        self.name = name

    def get_sound(self):
        return "Woof!"
```

and call the some function `make_sound()` on `Dog("Doggo")`:

```Bash
> mypy ducktyping.py
ducks_py3.py:28: error: Argument 1 to "make_sound" has incompatible type "Dog"; expected "Duck"
```

This happened because `mypy` looks and the type annotation,
sees that the expected type is a `Duck`, actual type is a `Dog` and evaluates it as incompatible.
OOP fans might come up with a fairly simple solution - inheritance (also called *nominal subtyping*).
This would work, but there is a bit simpler approach - **structural subtyping**.
Instead of designing pretty complicated class hierarchy
and then using the most abstract class as the type annotation,
we can define a **protocol** - a class that just states all the attributes and methods we expect.
This is the equivalent of static duck typing.
Let's add an `AnimalProtocol` class,
which specifies all the attributes and methods we will be working with:

```Python
"""Python3 ducktyping demo"""
from typing_extensions import Protocol

class AnimalProtocol(Protocol):
    # notice the type annotation for `name` attribute
    name: str

    def make_sound(self) -> str:
        # we can omit the body of this method
        ...

class Duck(object):
    def __init__(self, name: str) -> None:
        self.name = name

    def make_sound(self) -> str:
        return "Quack!"

class Dog(object):
    def __init__(self, name: str) -> None:
        self.name = name

    def make_sound(self) -> str:
        return "Woof!"

def make_sound(animal: AnimalProtocol) -> None:
    """Print the sound of the animal."""
    print(f"{animal.name} does '{animal.make_sound()}'")

if __name__ == "__main__":
    make_sound(Duck("Duck named Bob"))
    make_sound(Dog("Doggo"))

```

Notice that we changed the type annotation of `make_sound()` function:

```Python
# previous declaration:
def make_sound(animal: Duck) -> None:

# current declaration:
def make_sound(animal: AnimalProtocol) -> None:
```

Running `mypy` on this code will not result in any error messages.

## Writing Python2 compatible protocols

Since `Protocols` were introduced in Python3,
we have to come up with a workaround when doing structural subtyping in Python2.
Luckily, this can be done using abstract metaclasses:

```Python
"""AnimalProtocol in Python2"""
from abc import ABCMeta, abstractmethod

@add_metaclass(ABCMeta)
class AnimalProtocol(object):
    """Animal Protocol in Python2."""

    name = None  # type: str

    def make_sound(self):
        # type: () -> str
        return
```
