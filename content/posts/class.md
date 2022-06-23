+++
title = "Advanced python"
description = "Data classes, property setters and getters"
date = 2020-05-28
+++

_**Disclaimer: Everything below is typed Python**_

## Dataclasses

The following snippet, desugars to

_Snippet 1:_

```python
@dataclass
class Person:
    name: str
    age: float
```

_Snippet 2:_

```python
class Person:
    def __init__(self, name: str, age: float):
        self.name = name
        self.age = age
```

There are two patterns to create dataclasses

1. Immutable class
2. Mutable class

### Immutable dataclasses

_Snippet 3:_

```python
@dataclass(frozen=True)
class Person:

    name: str
    age: float

    @classmethod
    def from_birthyear(cls, name: str, birth_year: int) -> Person:
        return cls(name, date.today().year - birth_year)

    @staticmethod
    def from_fathers_age(
        name: str, fathers_age: int, diff_from_fathers_age: int
    ) -> Person:
        return Person(name, fathers_age - diff_from_fathers_age)

    @staticmethod
    def nationality() -> str:
        """ All are japanese """
        return "Japan"


print(Person.nationality())

print(Person.from_fathers_age("Siva", 60, 32))

me = Person.from_birthyear("Siva", 1992)
print(me.age)
```

### Mutable dataclasses

Field names can start with underscore, so that `@property` getters can use the method names without underscore at the beginning. Private fields are to be started with double underscores. `name` does not work as a field name, but `_name` is fine. We can then create `@property` with method name as `name`. It's a good practice to **access the fields using property getters**.

_Snippet 4:_

```python
@dataclass
class MutablePerson:
    """
 Another note:
 _name works but not name, because I created a property name "name" was created for it
 """

    _name: str
    __age: float

    @classmethod
    def from_birthyear(cls, name: str, birth_year: int) -> Person:
        return cls(name, date.today().year - birth_year)

    @staticmethod
    def from_fathers_age(
        name: str, fathers_age: int, diff_from_fathers_age: int
    ) -> Person:
        return Person(name, fathers_age - diff_from_fathers_age)

    @staticmethod
    def nationality() -> str:
        """ All are japanese """
        return "Japan"

    @property
    def name(self) -> str:
        return self._name

    @name.setter
    def name(self, name_value) -> None:
        self._name = name_value


print("-" * 100)
print("mutable person")
print(MutablePerson("Siva", 28)._MutablePerson__age)
print("-" * 100)

print(MutablePerson("Noona", 28)._name)

print(MutablePerson.from_birthyear("Siva", 1992).name)
print(
    MutablePerson.from_birthyear("Siva", 1992)._name
)  # generally not preferred, you create a property getter like above

siva = MutablePerson("Siva", 28)
print(siva.name)
siva.name += "ram"
print(siva.name)

print("-" * 100)
```

Another example

_Snippet 5:_

```python
@dataclass
class Employee:

    _name: str
    _company: str
    _retired: str = "NO"

    @property
    def company(self):
        return self._company

    @company.setter
    def company(self, value):
        self._company = value


e = Employee("Siva", "Rakuten")
print("Company name is :", e.company)
e.company = "Google"
print("Company name is :", e.company)
```

## `@staticmethod` vs `@classmethod`

We can use either annotation to create a class from their methods. `@classmethod` takes class as first argument. As a rule of thumb, if your methods accesses other variables/methods in the class, we can use `@classmethod`. `@staticmethod` does not access any of the classes's properties. We can also use `@staticmethod` to create a class like _Snipprt 3:_.

## Typing in Python

If we want to use class types inside the class, we need to use this import at the top of the file. This will become default in Python 4.

```python
from __future__ import annotations
```

For example, the return type of `from_birthyear` is `Person` but this only works when we import `annotations`

```python
@dataclass(frozen=True)
class Person:

 ...

    @classmethod
    def from_birthyear(cls, name: str, birth_year: int) -> Person:
        return cls(name, date.today().year - birth_year)
 
 ...

```

## Normal classes

For arbitrary initialization of class objects, we always use `__init__`. We cannot create `_randomly_fill` when we use `@dataclass`.

```python
from dataclasses import dataclass

from enum import Enum
from typing import NamedTuple, List


class Cell(str, Enum):
    EMPTY = " "
    BLOCKED = "X"
    START = "S"
    GOAL = "G"
    PATH = "*"


"""
# Alternate version
class MazeLocation(NamedTuple):
 row: int
 column: int
"""


@dataclass
class MazeLocation:

    row: int
    column: int


class Maze:
    def __init__(
        self,
        rows: int = 10,
        columns: int = 10,
        sparseness: float = 0.2,
        start: MazeLocation = MazeLocation(0, 0),
        goal: MazeLocation = MazeLocation(9, 9),
    ) -> None:
        self._rows = rows
        self._columns = columns
        self.start: MazeLocation = start
        self.goal: MazeLocation = goal
        # fill grid with empty cells
        self._grid: List[List[Cell]] = [
            [Cell.EMPTY for c in range(columns)] for r in range(rows)
        ]
        self._randomly_fill(rows, columns, sparseness)
        self._grid[start.row][start.column] = Cell.START
        self._grid[goal.row][goal.column] = Cell.GOAL

    def _randomly_fill(self, rows: int, columns: int, sparseness: float):
        for row in range(rows):
            for column in range(columns):
                if random.uniform(0, 1.0) < 0.2:
                    self._grid[row][column] = Cell.BLOCKED

    def __str__(self) -> str:
        output: str = ""
        for row in self._grid:
            output += "".join([c.value for c in row]) + "\n"
        return output


maze = Maze()
print(maze)

```

Another small note about property in Python class.

- If it's a property, we can call the property directly without parans

Example

```python
class Stack(Generic[T]):
    def __init__(self) -> None:
        self._container = []

    def empty(self) -> bool:
        return not self._container

    ...

a = Stack[int]()
a.push(10)
print(a.empty())


class StackWithEmptyProperty(Generic[T]):
    def __init__(self) -> None:
        self._container = []

    @property
    def empty(self) -> bool:
        return not self._container

    ...

a = Stack[int]()
a.push(10)
print(a.empty)
```
