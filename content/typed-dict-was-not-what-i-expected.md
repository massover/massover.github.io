Title: TypedDict was not what I expected
Date: 2021-02-07 10:53am
Category: python

## Example TypedDict

```python
try:
    from typing import TypedDict
except ImportError:
    from typing_extensions import TypedDict

class User(TypedDict):
    first_name: str
    last_name: str
    email: str
```

## I expected it to work like a dataclass

If I didn't provide class attributes to the constructor, I expected
it to error at runtime.

```python
In [6]: class User(TypedDict):
   ...:     first_name: str
   ...:     last_name: str
   ...:     email: str
   ...: 

In [7]: user = User(first_name="John", last_name="Smith")

In [8]: user['first_name']
Out[8]: 'John'
```

At first this seemed unhelpful. However the issue here is mostly with my understanding and usage of type hinting. It doesn't
need to error at runtime. That's the purpose of catching this via typing hinting using mypy.

```python
# main.py
from user import User

def main():
    user = User(
        first_name="John",
        last_name="Smith"
    )
```
---

```bash
mypy main.py

main.py:11: error: Missing key 'email' for TypedDict "User"
Found 1 error in 1 file (checked 1 source file)
```

## I expected it to behave like a normal class

I wanted to add a method to the class.

```python
In [10]: class User(TypedDict):
    ...:     first_name: str
    ...:     last_name: str
    ...:     email: str
    ...:     def get_full_name(self):
    ...:         return f"{self.first_name} {self.last_name}"
    ...: 

In [11]: user = User(first_name="John", last_name="Smith", email="john.smith@example.com")

In [12]: user.get_full_name()
---------------------------------------------------------------------------
AttributeError                            Traceback (most recent call last)
<ipython-input-12-d64390302591> in <module>
----> 1 user.get_full_name()

AttributeError: 'dict' object has no attribute 'get_full_name'
```

The [docs](https://www.python.org/dev/peps/pep-0589/#class-based-syntax) talk about the restrictions of the
class based syntax, including

> Methods are not allowed, since the runtime type of a TypedDict object will always be just dict (it is never a subclass of dict).

Now I understand why the docs focus on this example, and only show the callable class example later. 

```python
movie: Movie = {'name': 'Blade Runner',
                'year': 1982}
```

As in the name, `TypedDict` has its use for adding type hints to dicts.

