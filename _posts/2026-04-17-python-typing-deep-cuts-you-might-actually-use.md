---
layout: post
title: "Python typing deep cuts you might actually use"
date: 2026-04-17
---

# TypeVar 

A type variable is a placeholder in type expressions that you can use when you don't know exactly what type will be used ahead of time. For example, 
```python
def first(things: list[T]) -> T:
    return things[0]
```
This is a function that takes a list and returns the first item in that list. It could be list of anything, such as `list[int]`, `list[str]` etc. Using a type variable `T` in this function signature tells us, "whatever is inside the list, I'm going to return one thing of that type."

Before Python 3.12, you had to define type variables ahead of time, like this
```python
from typing import TypeVar

T = TypeVar("T")


def first(things: list[T]) -> T:
    return things[0]
```
Now, you can define them in the function signature
```python
def first[T](things: list[T]) -> T:
    return things[0]
```
You can restrict which types are allowed in the type variable with bounds and constraints. A bound means, any subtype of x is allowed in this type variable. For example
```python
from collections.abc import Hashable


def dedup[T: Hashable](items: list[T]) -> list[T]:
    return list(set(items))
```
For `dedup()` to work, items in the list must be hashable (or `set()` would complain). So, we can demand that whatever type `T` represents must be a subtype of `Hashable`. The syntax is, you specify the bound type after a colon inside the square brackets.

Constraints are similar, except the mystery type must be exactly one of the constraint types, not a subtype of them. For example
```python
def double[T: int | float](x: T) -> T:
    return x + x
```
You might be suprised that the type checker doesn't allow this. Why? Because `bool` is a subtype of `int`. If you pass a boolean to this function, `T` represents `bool`, but adding two booleans give you an `int`, not a `bool`. Therefore we need a stricter rule: `T` must be exactly `int` or `float`, not a subtype. That's what constraints are for
```python
def double[T: (int, float)](x: T) -> T:
    return x + x
```
The syntax is just like bounds, except the constraint types go in a tuple. Note that you can't do constraints with only one type
```python
def double[T: (int,)](x: T) -> T:
    return x + x
```
Because that would be equivalent to a normal type annotation
```python
def double(x: int) -> int:
    return x + x
```

# ParamSpec
Imagine a decorator used to add tracing
```python
def trace(f):
    def wrapped(*args, **kwargs):
        with tracer.trace():
            return f(*args, **kwargs)
    return wrapped
```
Decorators are functions that take another function as an argument, and return a new function. How do we add type annotations to this?
```python
from typing import Any, Callable


def trace(f: Callable[..., Any]) -> Callable[..., Any]:
    def wrapped(*args: Any, **kwargs: Any):
        with tracer.trace():
            return f(*args, **kwargs)
    return wrapped
```
This tells the type checker, "trace() accepts one argument, which is a callable with any call signature, and returns another callable with any call signature". The problem is that when you use this, it completely hides any real type information on the function `f`
```python
@trace
def send(message: str) -> bool:
    ...
```
Decorating `send()` produced a new function `wrapped()`. Based on our type annotations, the type checker thinks `wrapped()` is a callable with any call signature. But wait, `send()` has a very specific call signature. It takes a `str` and returns a `bool`. And we know that the new function `wrapped()` should follow the same rules. Now, every time we call `send()`, the type checker will treat it as a callable that takes any arguments and returns anything. All of the information in our type annotations is lost! That's where `ParamSpec` comes in. `ParamSpec` is a special kind of type variable that represents the type of function parameters.
```python
def trace[T, **P](f: Callable[P, T]) -> Callable[P, T]:
    def wrapped(*args: P.args, **kwargs: P.kwargs) -> T:
        with tracer.trace():
            return f(*args, **kwargs)
    return wrapped
```
The square bracket syntax `[T, **P]` declares a type variable `T` and a ParamSpec `P` we can use for this function. `P` has attributes called args and kwargs that represent the type of whatever positional and keyword arguments are represented by `P`. Now, the type checker knows a lot more about `trace()`. It knows that `trace()` accepts a callable, with parameters `P` and return type `T`, and returns another callable with parameters `P` and return type `T`. 

# Never and NoReturn
How do you annotate a function that never returns anything? For example
```python
def handle_exception(e: Exception):
    match e:
        case ValueError():
            raise HTTPError(400, str(e))
        case _:
            raise HTTPError(500)
```
This function always raises an exception and therefore never returns something. Python 3.6.2 added the `NoReturn` type to express that
```python
from typing import NoReturn


def handle_exception(e: Exception) -> NoReturn:
    match e:
        case ValueError():
            raise HTTPError(400, str(e))
        case _:
            raise HTTPError(500)
```
Because of how type checkers work, you can also use it to specify an argument should never be passed, for example
```python
from typing import overload, Literal, NoReturn, TextIO, BinaryIO


@overload
def open_file(path: str, mode: Literal["r"], encoding: str = "utf-8") -> TextIO: ...

@overload
def open_file(path: str, mode: Literal["rb"], encoding: NoReturn = ...) -> BinaryIO: ...

def open_file(path, mode, encoding=None):
    ...
```
The type checker would dish out a red squiggle if you passed any value to the encoding parameter when using "rb" mode. Once people started doing this, the name `NoReturn` didn't make sense anymore, so Python 3.11 added `Never`. `Never` is exactly the same as `NoReturn`, it just has a more widely-applicable name. 
```python
from typing import Never 


@overload
def open_file(path: str, mode: Literal["r"], encoding: str = "utf-8") -> TextIO: ...


@overload
def open_file(path: str, mode: Literal["rb"], encoding: Never = ...) -> BinaryIO: ...


def open_file(path, mode, encoding=None):
    ...
```
This comes in handy if you want the type checker to yell at people for passing a certain argument.

# Final
Final is a generic type used to forbid people from reassigning a variable. It's similar to `const` in TypeScript or Go
```python
from typing import Final


port: Final[int] = 8000

...

port = 8001 # red squiggle
```
You can also use it in class variables to stop subclasses from changing the value
```python
class Server:
    port: Final[int] = 8000


class FancyServer(Server):
    port = 8001 # red squiggle
```

# @final
You can use the @final decorator on a class to forbid inheriting from it, or use it on methods to forbid subclasses from overriding them
```python
from typing import final


@final
class ChuckNorris:
    ...


class ChuckNorrisJr(ChuckNorris): # red squiggle
    ...


class Fruit:
    @final
    def eat(): ...


class Banana(Fruit):
    def eat(): ... # red squiggle

```
You can use it when overriding the class or method would cause a real problem, or to formally express design intentions.

# TypeIs
There are some types that the type checker doesn't know how to verify on its own. For example
```python
from typing import Any, Literal


type ListOfBananas = list[Literal["banana"]]


def go_bananas(l: ListOfBananas) -> None:
    print("BANANAS!!!")
    for banana in l:
        print(banana)


def maybe_go_bananas(l: list[Any]) -> None:
    if isinstance(l, ListOfBananas): 
        go_bananas(l)
    else:
        pass
```
This doesn't work because `isinstance()` can't check values inside a list, nor can it check if a given string matches the "banana" literal. You'd have to write something like this
```python
from typing import Any, Literal


type ListOfBananas = list[Literal["banana"]]


def go_bananas(l: ListOfBananas) -> None:
    print("BANANAS!!!")
    for banana in l:
        print(banana)


def maybe_go_bananas(l: list[Any]) -> None:
    if isinstance(l, list) and all(item == "banana" for item in l):
        go_bananas(l)
    else:
        pass
```
That looks a lot less fun, you would have to rewrite it everywhere you're checking for a list of bananas, and the type checker still wouldn't actually narrow the type to `ListOfBananas`. You can use `TypeIs` to write a custom type guard function instead
```python
from typing import Any, Literal


type ListOfBananas = list[Literal["banana"]]


def go_bananas(l: ListOfBananas) -> None:
    print("BANANAS!!!")
    for banana in l:
        print(banana)


def is_list_of_bananas(l: Any) -> TypeIs[ListOFBananas]:
    return isinstance(l, list) and all(item == "banana" for item in l):


def maybe_go_bananas(l: list[Any]) -> None:
    if is_list_of_bananas(l)
        go_bananas(l)
    else:
        pass
```
Functions annotated to return `TypeIs` must take exactly one argument and return a boolean. The `TypeIs` return type annotation tells the type checker, "when this function returns true, the argument has this type."