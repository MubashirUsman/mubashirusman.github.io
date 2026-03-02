---
layout: post
title:  "Python programming tips using standard library"
date:   2025-12-25 00:30:20 +0000
categories: programming python
---
# **Python tips from standard library for my reference** 

## Dataclasses
Dataclasses is essentially a code generator. It helps to avoide writing boilerplate and repeating code. Following are important use cases.
1. We don't need to write _init_ method if class is an instance of dataclass. For example:
```
@dataclass  
class Circle():  
    x: int = 0  
    y: int = 0  
    radius: int = 1
```
Even though we defined class level variables, they will act as if they were instance variables.  

2. Also it implements _repr_ method automatically. And if instance variables needs to be made immutable then we can pass _frozen_ parameter to dataclass. This will also implement _hash_ method and make the objects hashable, which means we can use them as keys in a dictionary.
```@dataclass(frozen=True)```

3. We can set `order=True` to implement equality or less-than/greater-than methods.

## Type Hinting
1. Even though Cpython completely ignores variable types set by type-hinting, but type-hinting is useful in many other ways. Like for documentation, and libraries such as Pydantic and dataclasses uses type-hinting.

Example:
```
def func(a: dict, b: list, c: bool = True) -> str:
    return f"a= {a}, b={b}"
```

2. As a parameter in Python can take different argument types, we can do like this.  
```
from typing import Union
def funcm(a: Union[str, int], b: int) -> Union[str, int]:
    return a*b

OR

def funcm(a: str | int, b: int) -> str | int:
    return a*b
```
3. If an argument could be passed as a specific type or could be None but is NOT optional, one can use Optional to specify this.
```
from typing import Optional
def funco(a Optional[int]) -> None:
    pass
```
4. For containers like lists/tuples/dicts, we can use generics from typing module
```
from typing import List
def funcg(a: List[float]) -> List[int]:
    return [int(i) for i in a]
```
5. OR for functions and generators
```
from typing import Callable, Any, Sequence, Iterator
def funcf(func: Callable[[Any], Any], sequence: Sequence[Any]) -> Iterator[str]:
    for i in sequence:
        yield str(func(i))
```
NOTE: From Python 3.9 onwards, many generics are being deprecated in `typing` module and moved to other modules like `collections.abc`. 