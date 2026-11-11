# Python Interview Answers (5-8 Years, Full Stack)

## Part 1: Core Python Questions

### 1. What is the difference between a list, tuple, set, and dict, and when would you pick each for performance?

**Short answer:** A list is an ordered, changeable collection that allows duplicates. A tuple is an ordered, unchangeable collection. A set is an unordered collection of unique items with O(1) lookup. A dict stores key–value pairs with O(1) lookup by key.

**Real-life example:** A list is a shopping cart, a tuple is a printed bill, a set is a party guest list (no repeats), and a dict is a phone contact book (name → number).

```python
cart = ["Shoes", "Socks", "Socks"]          # list: order + duplicates
location = (12.97, 77.59)                   # tuple: fixed value, hashable
emails = {"a@x.com", "b@x.com"}             # set: unique, O(1) 'in'
product = {"sku": "S1", "price": 999}       # dict: lookup by key

print("a@x.com" in emails)     # True, fast hash lookup
print(product["price"])        # 999

# Real use: avoid N+1 queries in Django with a dict
customer_map = {c.id: c for c in Customer.objects.filter(id__in=ids)}
customer = customer_map[order.customer_id]
```

**Interview tip:** Frequent membership checks → set. Lookup by ID → dict. Fixed data or dict key → tuple. Ordered, changing sequence → list.

### 2. Explain generators and yield. When would you use one instead of returning a list?

**Short answer:** A generator is a function that returns values one at a time using `yield`, pausing between each, instead of building the whole result in memory.

**Real-life example:** A list is a bakery baking 10,000 cakes upfront. A generator bakes one cake only when a customer asks: less space, and the first customer is served immediately.

```python
def count_up(n):
    for i in range(n):
        yield i

gen = count_up(3)
print(next(gen)) # 0
print(next(gen)) # 1

# Real use: read a huge file in constant memory
def read_logs(path):
    with open(path) as f:
        for line in f:
            yield line.strip()

total = sum(x * x for x in range(10_000_000))    # generator expression
```

**Interview tip:** Use a generator for large, streamed, or infinite data. Use a list when you need indexing, `len()`, or more than one pass.

### 3. How do decorators work? Write one that logs execution time or retries a failing call.

**Short answer:** A decorator is a function that wraps another function to add behaviour without changing its code. `@timer` above a function means `func = timer(func)`.

**Real-life example:** Gift wrapping: the gift stays the same, the wrapping adds something around it. Django's `@login_required` is a guard standing in front of a view.

```python
import time, functools

def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.3f}s")
        return result
    return wrapper

def retry(times=3, delay=1, exceptions=(Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, times + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions:
                    if attempt == times:
                        raise
                    time.sleep(delay * attempt)  # simple backoff
        return wrapper
    return decorator

@timer
@retry(times=3, exceptions=(ConnectionError,))
def call_payment_api():
    ...
```

### 4. What is the GIL, and how does it affect threading vs. multiprocessing vs. asyncio?

**Short answer:** The Global Interpreter Lock is a lock in CPython that allows only one thread to execute Python bytecode at a time.

**Real-life example:** A kitchen with many cooks but one knife. Cooks waiting for water to boil (I/O) share the knife well; for pure chopping (CPU work) extra cooks don't help. Multiprocessing = separate kitchens, each with its own knife. Asyncio = one efficient cook switching between dishes while each simmers.

CPU-bound (image resize, heavy maths) → multiprocessing. I/O-bound (API calls, DB) → threading. Thousands of I/O tasks → asyncio.

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import asyncio

with ThreadPoolExecutor(max_workers=10) as ex:      # I/O-bound
    results = list(ex.map(fetch_url, urls))

if __name__ == "__main__":
    with ProcessPoolExecutor() as ex:               # CPU-bound
        results = list(ex.map(heavy_calc, numbers))

async def main():                                   # many I/O tasks
    await asyncio.gather(*(fetch_async(u) for u in urls))
asyncio.run(main())
```

**Interview tip:** NumPy releases the GIL inside C code. Python 3.13 added an experimental free-threaded (no-GIL) build, and 3.14 made it officially supported (still optional, not the default).

### 5. Explain *args / **kwargs, mutable default arguments, and why def f(x=[]) is a bug.

**Short answer:** `*args` collects extra positional arguments into a tuple. `**kwargs` collects extra keyword arguments into a dict. A mutable default like `[]` is created once, when the function is defined, and shared across all calls.

**Real-life example:** `*args` is an unlabelled bag, `**kwargs` is a labelled bag. The mutable default bug is a restaurant giving every customer the same plate, so the next customer gets leftovers.

```python
def order(name, *items, **options):
    print(items)    # ('Pizza', 'Coke')
    print(options) # {'express': True}

order("Asha", "Pizza", "Coke", express=True)

# Bug
def add(item, items=[]):
    items.append(item)
    return items
add(1) # [1]
add(2) # [1, 2] <- shared list!

# Fix
def add(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### 6. Shallow vs. deep copy; is vs. ==

**Short answer:** A shallow copy creates a new outer object but shares inner objects. A deep copy copies everything recursively. `==` compares values; `is` checks whether both names point to the same object.

**Real-life example:** Shallow copy is photocopying only a folder's cover (papers inside are shared). Deep copy photocopies every page. `==` means twins; `is` means the same person.

```python
import copy

a = [[1, 2], [3]]
s = copy.copy(a)
d = copy.deepcopy(a)
a[0].append(99)
print(s) # [[1, 2, 99], [3]]    shared inner list
print(d) # [[1, 2], [3]]        independent

x, y = [1, 2], [1, 2]
print(x == y) # True
print(x is y) # False

if value is None:    # use 'is' only for None / True / False
    ...
```

### 7. Context managers: how does with work, and how do you write one?

**Short answer:** A context manager handles setup and cleanup around a block of code, even if an exception occurs. `with` calls `__enter__` at the start and `__exit__` at the end.

**Real-life example:** A hotel stay: check-in gives you the key (`__enter__`), check-out happens automatically when you leave (`__exit__`), even during a fire alarm.

```python
class Transaction:
    def __init__(self, conn):
        self.conn = conn
    def __enter__(self):
        self.conn.begin()
        return self.conn
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.conn.rollback()
        else:
            self.conn.commit()
        return False           # don't swallow the exception

from contextlib import contextmanager
import time

@contextmanager
def timer(label):
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label}: {time.perf_counter() - start:.2f}s")

with timer("report"):
    build_report()
```

**Interview tip:** Django's `with transaction.atomic():` is exactly this pattern.

### 8. What are dunder methods? Implement __eq__ / __hash__ for a custom class.

**Short answer:** Dunder methods (`__init__`, `__str__`, `__len__`, `__eq__`...) let a class plug into Python's built-in behaviour. Rule: if `a == b`, then `hash(a) == hash(b)`. Defining `__eq__` alone makes the object unhashable.

**Real-life example:** Two Aadhaar cards with the same number belong to the same person, even if the photo differs. `__eq__` compares the number; `__hash__` uses the same number.

```python
class Product:
    def __init__(self, sku, name, price):
        self.sku, self.name, self.price = sku, name, price

    def __eq__(self, other):
        if not isinstance(other, Product):
            return NotImplemented
        return self.sku == other.sku

    def __hash__(self):
        return hash(self.sku)

    def __repr__(self):
        return f"Product({self.sku!r}, {self.name!r})"

    def __lt__(self, other):          # enables sorted()
        return self.price < other.price

p1 = Product("S1", "Shoes", 2000)
p2 = Product("S1", "Shoes v2", 2000)
print(p1 == p2)        # True
print(len({p1, p2}))   # 1
```

**Interview tip:** `@dataclass(frozen=True)` generates `__eq__` and `__hash__` for you. Hash only on fields that never change.

### 9. How does Python manage memory (reference counting, garbage collection)?

**Short answer:** CPython frees an object when its reference count drops to zero. A generational garbage collector cleans up reference cycles that counting can't handle.

**Real-life example:** A library book tracks how many people hold it; at zero it's returned. If two people each say "the other has my book" (a cycle), the librarian (GC) periodically finds and reclaims it.

```python
import sys, gc

a = []
print(sys.getrefcount(a))

class Node:
    def __init__(self):
        self.ref = None

x, y = Node(), Node()
x.ref, y.ref = y, x     # cycle
del x, y                # refcounts never reach 0
print(gc.collect())     # GC frees the cycle

import tracemalloc      # finding real leaks
tracemalloc.start()
```

**Interview tip:** Real-world "leaks" are usually references you keep alive (e.g. a global cache that grows forever). Use `lru_cache(maxsize=...)` or `weakref`.

### 10. Type hints, dataclasses, and Pydantic: where does each fit?

**Short answer:** Type hints describe expected types but are not enforced at runtime. Dataclasses auto-generate `__init__`, `__repr__`, `__eq__`, with no validation. Pydantic validates and converts data at runtime.

**Real-life example:** Type hints are a label on a box. A dataclass is a machine that builds the box. Pydantic is a security checkpoint that inspects what's inside.

```python
def total(price: float, qty: int) -> float:     # hints only
    return price * qty

from dataclasses import dataclass, field

@dataclass
class Item:
    name: str
    qty: int = 1
    tags: list[str] = field(default_factory=list)

from pydantic import BaseModel, Field

class User(BaseModel):
    name: str
    age: int = Field(ge=18)

User(name="Asha", age="25")     # age -> 25
User(name="Asha", age="abc")    # ValidationError
```

**Interview tip:** Type hints everywhere, dataclasses for internal objects, Pydantic wherever outside data enters (FastAPI request bodies, configs).

## Part 2: Basics and Data Types

### 11. What is the difference between mutable and immutable types?

**Short answer:** Mutable objects can be changed after creation (list, dict, set). Immutable objects cannot (int, float, str, tuple, frozenset); any "change" creates a new object.

**Real-life example:** A whiteboard (mutable) can be edited in place; a printed certificate (immutable) must be reprinted to change.

```python
nums = [1, 2]
nums.append(3)          # same object, changed

name = "asha"
upper = name.upper() # new string; 'name' unchanged
print(id(name) == id(upper)) # False
```

### 12. Why are strings immutable in Python?

**Short answer:** Immutability makes strings hashable (usable as dict keys), thread-safe, and allows Python to cache/intern them safely.

**Real-life example:** A dict key that could change after insertion would be "lost" in the wrong hash bucket, like a locker whose number keeps changing.

```python
s = "hello"
# s[0] = "H"         # TypeError
s = "H" + s[1:]      # build a new string instead
cache = {"user:1": "Asha"}   # works because str is hashable
```

### 13. What is the difference between append(), extend(), and insert()?

**Short answer:** `append(x)` adds one item at the end. `extend(iterable)` adds each item of an iterable. `insert(i, x)` adds an item at a specific index (O(n)).

```python
a = [1, 2]
a.append([3, 4])     # [1, 2, [3, 4]]
b = [1, 2]
b.extend([3, 4])     # [1, 2, 3, 4]
c = [1, 3]
c.insert(1, 2)       # [1, 2, 3]
```

### 14. What is the difference between remove(), pop(), and del?

**Short answer:** `remove(x)` deletes the first matching value. `pop(i)` deletes by index and returns the item (last by default). `del` deletes by index/slice or deletes the variable itself.

```python
a = [10, 20, 30, 20]
a.remove(20)     # [10, 30, 20]
x = a.pop()      # x = 20, a = [10, 30]
del a[0]         # [30]
del a            # variable removed
```

### 15. What are list, dict, and set comprehensions?

**Short answer:** A compact syntax to build a collection from an iterable, optionally with a filter. Usually faster and more readable than a loop with `append`.

```python
squares = [x * x for x in range(5)]                  # list
evens = [x for x in range(10) if x % 2 == 0]
price_map = {p.sku: p.price for p in products}       # dict
cities = {o.city for o in orders}                    # set (unique)
```

### 16. What is the difference between range() and a list?

**Short answer:** `range` is a lazy, immutable sequence that computes numbers on demand and uses constant memory. A list stores every element.

```python
import sys
print(sys.getsizeof(range(1_000_000)))          # ~48 bytes
print(sys.getsizeof(list(range(1_000_000))))    # ~8 MB
print(500 in range(1_000_000))                  # O(1) for range
```

### 17. How does slicing work? What does lst[::-1] do?

**Short answer:** Slicing uses `[start:stop:step]`; stop is excluded, negative values count from the end. `[::-1]` returns a reversed copy.

```python
a = [0, 1, 2, 3, 4, 5]
a[1:4]    # [1, 2, 3]
a[:3]     # [0, 1, 2]
a[-2:]    # [4, 5]
a[::2]    # [0, 2, 4]
a[::-1]   # [5, 4, 3, 2, 1, 0]
```

### 18. What is None, and how is it different from 0, "" or False?

**Short answer:** `None` is a singleton meaning "no value". `0`, `""` and `False` are real values that happen to be falsy. Check `None` with `is None`.

**Real-life example:** An empty form field (`None`) is different from a field where someone typed `0`.

```python
def apply_discount(price, discount=None):
    if discount is None:       # not provided
        discount = 5
    return price - discount

apply_discount(100, 0)   # 100 - 0 works; 'if not discount' would wrongly use 5
```

### 19. What are truthy and falsy values?

**Short answer:** In a boolean context, `None`, `False`, `0`, `0.0`, `""`, `[]`, `{}`, `set()` and `range(0)` are falsy; almost everything else is truthy.

```python
orders = []
if not orders:
    print("No orders yet")

name = input_name or "Guest"     # fallback using falsy check
```

### 20. What is the difference between sort() and sorted()?

**Short answer:** `list.sort()` sorts in place and returns `None` (lists only). `sorted()` works on any iterable and returns a new list. Both accept `key` and `reverse`.

```python
nums = [3, 1, 2]
nums.sort()                 # nums -> [1, 2, 3]
print(sorted("dcba"))       # ['a', 'b', 'c', 'd']
emps = sorted(employees, key=lambda e: e["salary"], reverse=True)
```

## Part 3: Functions

### 21. What is a lambda function? When should you avoid it?

**Short answer:** A lambda is a small anonymous single-expression function. Avoid it when logic is complex, needs a docstring, or is reused; use `def` instead.

```python
square = lambda x: x * x
users.sort(key=lambda u: u["age"])          # good use: short key function
```

**Interview tip:** Assigning a lambda to a name (like `square` above) is discouraged by PEP 8; use `def`.

### 22. Explain map(), filter(), and reduce().

**Short answer:** `map` applies a function to every item. `filter` keeps items where the function returns True. `reduce` combines items into one value. `map`/`filter` return lazy iterators.

```python
from functools import reduce
prices = [100, 200, 300]

with_tax = list(map(lambda p: p * 1.18, prices))   # [118.0, 236.0, 354.0]
expensive = list(filter(lambda p: p > 150, prices)) # [200, 300]
total = reduce(lambda a, b: a + b, prices)         # 600
```

**Interview tip:** Comprehensions are often more Pythonic: `[p * 1.18 for p in prices]`.

### 23. What is a closure?

**Short answer:** A closure is an inner function that remembers variables from its enclosing function even after that function has finished.

**Real-life example:** A rubber stamp pre-set with a company name keeps stamping that name wherever it goes.

```python
def make_multiplier(n):
    def multiply(x):
        return x * n          # 'n' is remembered
    return multiply

double = make_multiplier(2)
print(double(5))   # 10
```

### 24. What is the LEGB rule?

**Short answer:** Python resolves names in this order: Local → Enclosing → Global → Built-in.

```python
x = "global"
def outer():
    x = "enclosing"
    def inner():
        x = "local"
        print(x)      # local
    inner()
outer()
print(len)            # built-in
```

### 25. What do global and nonlocal do?

**Short answer:** `global` lets a function rebind a module-level variable. `nonlocal` lets an inner function rebind a variable from its enclosing function.

```python
count = 0
def inc():
    global count
    count += 1

def counter():
    total = 0
    def add():
        nonlocal total
        total += 1
        return total
    return add
c = counter(); c(); print(c())   # 2
```

### 26. What is recursion, and what is Python's recursion limit?

**Short answer:** Recursion is a function calling itself with a base case to stop. Default limit is about 1000 frames (`sys.getrecursionlimit()`); Python has no tail-call optimisation.

```python
def factorial(n):
    if n <= 1:          # base case
        return 1
    return n * factorial(n - 1)

import sys
print(sys.getrecursionlimit())   # 1000
```

**Interview tip:** For deep recursion prefer an iterative solution or an explicit stack.

### 27. What are first-class functions?

**Short answer:** Functions are objects: they can be assigned to variables, passed as arguments, returned from functions, and stored in data structures.

```python
def email(msg): print("Email:", msg)
def sms(msg): print("SMS:", msg)

notifiers = {"email": email, "sms": sms}
notifiers["sms"]("Order shipped")    # dispatch table
```

### 28. What are keyword-only and positional-only arguments?

**Short answer:** Parameters after `*` must be passed by keyword. Parameters before `/` must be passed positionally (Python 3.8+).

```python
def transfer(amount, /, *, to_account, note=""):
    ...

transfer(500, to_account="ACC1")         # OK
# transfer(amount=500, to_account="ACC1") # TypeError (positional-only)
# transfer(500, "ACC1")                    # TypeError (keyword-only)
```

**Interview tip:** Keyword-only args make calls self-documenting and prevent mix-ups.

### 29. What is functools.partial?

**Short answer:** `partial` creates a new function with some arguments of an existing function pre-filled.

```python
from functools import partial

def send(channel, message):
    print(f"[{channel}] {message}")

send_slack = partial(send, "slack")
send_slack("Deploy finished")    # [slack] Deploy finished
```

### 30. What does functools.lru_cache do?

**Short answer:** It memoises function results: repeated calls with the same arguments return the cached result. `maxsize` limits memory using Least-Recently-Used eviction. Arguments must be hashable.

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

print(fib(100))           # instant
print(fib.cache_info())
```


## Part 4: Object-Oriented Programming

### 31. What is the difference between a class and an object?

**Short answer:** A class is a blueprint; an object (instance) is a concrete thing built from it with its own data.

**Real-life example:** The house plan is the class; each actual house on the street is an object.

```python
class Car:
    def __init__(self, brand):
        self.brand = brand

c1 = Car("Tata")      # object 1
c2 = Car("Mahindra")  # object 2
```

### 32. @staticmethod vs @classmethod vs instance method

**Short answer:** An instance method gets `self` and works on object data. A `classmethod` gets `cls` and works on the class (often alternative constructors). A `staticmethod` gets neither; it's a utility grouped inside the class.

```python
class User:
    count = 0
    def __init__(self, name):
        self.name = name
        User.count += 1

    def greet(self):                      # instance
        return f"Hi {self.name}"

    @classmethod
    def from_email(cls, email):           # alternative constructor
        return cls(email.split("@")[0])

    @staticmethod
    def is_valid_email(email):            # utility
        return "@" in email

u = User.from_email("asha@x.com")
```

### 33. What is self, and why is it needed?

**Short answer:** `self` is the reference to the current instance, passed automatically as the first argument of instance methods. It lets methods access that object's attributes.

```python
class Account:
    def __init__(self, balance):
        self.balance = balance
    def deposit(self, amt):
        self.balance += amt

acc = Account(100)
acc.deposit(50)      # same as Account.deposit(acc, 50)
```

### 34. What is the difference between __init__ and __new__?

**Short answer:** `__new__` creates and returns the instance (a static method on the class). `__init__` initialises the already-created instance and returns `None`. Override `__new__` for immutables or singletons.

```python
class Singleton:
    _instance = None
    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

a, b = Singleton(), Singleton()
print(a is b)   # True
```

### 35. What is inheritance? What is multiple inheritance?

**Short answer:** Inheritance lets a child class reuse and extend a parent class. Multiple inheritance means a class has more than one parent (commonly used for mixins).

```python
class Notification:
    def send(self): print("Sending")

class EmailNotification(Notification):         # single
    def send(self):
        super().send()
        print("via email")

class LoggingMixin:
    def log(self, msg): print("LOG:", msg)

class SMSNotification(LoggingMixin, Notification):   # multiple
    pass
```

### 36. What is MRO, and how does super() use it?

**Short answer:** Method Resolution Order is the order Python searches classes for a method, computed with C3 linearisation. `super()` calls the next class in the MRO, not necessarily the direct parent.

```python
class A:
    def hi(self): print("A")
class B(A):
    def hi(self): print("B"); super().hi()
class C(A):
    def hi(self): print("C"); super().hi()
class D(B, C):
    def hi(self): print("D"); super().hi()

D().hi()             # D B C A
print(D.__mro__)     # D, B, C, A, object
```

### 37. What is polymorphism? Give an example with duck typing.

**Short answer:** Polymorphism means different objects respond to the same method call in their own way. Duck typing: if it has the method you need, you can use it, regardless of its class.

**Real-life example:** "If it walks like a duck and quacks like a duck, it's a duck."

```python
class UPI:
    def pay(self, amt): print(f"UPI paid {amt}")
class Card:
    def pay(self, amt): print(f"Card paid {amt}")

def checkout(method, amount):
    method.pay(amount)           # doesn't care about the class

for m in (UPI(), Card()):
    checkout(m, 500)
```

### 38. What is encapsulation? What do _var and __var mean?

**Short answer:** Encapsulation is bundling data with methods and restricting direct access. `_var` is a convention for "internal, don't touch". `__var` triggers name mangling to `_ClassName__var` to avoid accidental override in subclasses.

```python
class Wallet:
    def __init__(self):
        self._currency = "INR"    # internal by convention
        self.__balance = 0        # name-mangled

    def add(self, amt):
        if amt > 0:
            self.__balance += amt

w = Wallet()
# w.__balance            -> AttributeError
print(w._Wallet__balance) # still reachable: not truly private
```

### 39. What is an abstract class? How do you create one with abc?

**Short answer:** An abstract class can't be instantiated and forces subclasses to implement certain methods. Created with `ABC` and `@abstractmethod`.

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount): ...

class Razorpay(PaymentGateway):
    def charge(self, amount):
        return f"Charged {amount}"

# PaymentGateway() -> TypeError
Razorpay().charge(100)
```

### 40. What is @property, and why use it?

**Short answer:** `@property` exposes a method as an attribute, letting you add validation or computed values without changing how callers access it.

```python
class Product:
    def __init__(self, price):
        self.price = price

    @property
    def price(self):
        return self._price

    @price.setter
    def price(self, value):
        if value < 0:
            raise ValueError("Price can't be negative")
        self._price = value

    @property
    def price_with_gst(self):
        return round(self._price * 1.18, 2)

p = Product(100)
print(p.price_with_gst)   # 118.0
```

### 41. What are __slots__, and why use them?

**Short answer:** `__slots__` declares a fixed set of attributes, removing the per-instance `__dict__`. Saves memory and speeds attribute access when you create millions of objects; you can't add new attributes.

```python
class Point:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x, self.y = x, y

p = Point(1, 2)
# p.z = 3   -> AttributeError
```

### 42. What is the difference between __str__ and __repr__?

**Short answer:** `__str__` is a readable string for end users (`print`). `__repr__` is an unambiguous string for developers (debugging, shell), ideally valid code to recreate the object. If `__str__` is missing, `__repr__` is used.

```python
class Order:
    def __init__(self, oid, total):
        self.oid, self.total = oid, total
    def __str__(self):
        return f"Order #{self.oid}: ₹{self.total}"
    def __repr__(self):
        return f"Order(oid={self.oid!r}, total={self.total!r})"

o = Order(7, 499)
print(o)        # Order #7: ₹499
print([o])      # [Order(oid=7, total=499)]
```

### 43. What is a metaclass? Where is it used?

**Short answer:** A metaclass is the "class of a class": it controls how classes themselves are created. `type` is the default metaclass. Django's `Model` uses a metaclass to turn field declarations into DB columns.

```python
class RegistryMeta(type):
    registry = {}
    def __new__(mcs, name, bases, attrs):
        cls = super().__new__(mcs, name, bases, attrs)
        mcs.registry[name] = cls
        return cls

class Plugin(metaclass=RegistryMeta): pass
class CSVExport(Plugin): pass

print(RegistryMeta.registry)   # {'Plugin': ..., 'CSVExport': ...}
```

**Interview tip:** Prefer `__init_subclass__` or decorators for simple cases.

### 44. Composition vs. inheritance: which do you prefer?

**Short answer:** Inheritance is an "is-a" relationship; composition is a "has-a" relationship where an object holds other objects. Prefer composition for flexibility and looser coupling; use inheritance for true is-a hierarchies.

**Real-life example:** A car has an engine (composition); it isn't an engine.

```python
class Engine:
    def start(self): print("Engine on")

class Car:
    def __init__(self, engine):
        self.engine = engine         # composition: swappable
    def drive(self):
        self.engine.start()

Car(Engine()).drive()
```

### 45. Implement Singleton in two ways.

**Short answer:** A Singleton ensures only one instance of a class exists (e.g. config, DB pool).

```python
# Way 1: override __new__
class Config:
    _instance = None
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

# Way 2: decorator
def singleton(cls):
    instances = {}
    def get(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get

@singleton
class Logger: pass

print(Config() is Config(), Logger() is Logger())   # True True
```

**Interview tip:** The simplest Pythonic singleton is a module: modules are imported once.

## Part 5: Iterators and Advanced Concepts

### 46. What is the difference between an iterable and an iterator?

**Short answer:** An iterable can be looped over and returns an iterator via `iter()` (list, str, dict). An iterator produces items one by one via `next()` and remembers its position; it's exhausted after one pass.

**Real-life example:** A book is iterable; a bookmark moving through it is the iterator.

```python
nums = [1, 2, 3]         # iterable
it = iter(nums)          # iterator
print(next(it), next(it), next(it))    # 1 2 3
# next(it) -> StopIteration
```

### 47. How do you create a custom iterator?

**Short answer:** Implement `__iter__` (returns `self`) and `__next__` (returns the next item or raises `StopIteration`).

```python
class Countdown:
    def __init__(self, start):
        self.current = start
    def __iter__(self):
        return self
    def __next__(self):
        if self.current <= 0:
            raise StopIteration
        self.current -= 1
        return self.current + 1

for n in Countdown(3):
    print(n)    # 3 2 1
```

### 48. What does yield from do?

**Short answer:** `yield from` delegates to another iterable or generator, yielding all its values (and passing through sends/return values).

```python
def flatten(items):
    for x in items:
        if isinstance(x, list):
            yield from flatten(x)
        else:
            yield x

print(list(flatten([1, [2, [3, 4]], 5])))    # [1, 2, 3, 4, 5]
```

### 49. What is itertools? Name useful functions.

**Short answer:** A standard library module of fast, memory-efficient iterator tools.

```python
from itertools import chain, groupby, islice, product, count

list(chain([1, 2], [3]))              # [1, 2, 3]
list(islice(count(10), 3))            # [10, 11, 12]
list(product("AB", [1, 2]))           # [('A',1),('A',2),('B',1),('B',2)]

orders = sorted(orders, key=lambda o: o["city"])   # groupby needs sorted data
for city, group in groupby(orders, key=lambda o: o["city"]):
    print(city, len(list(group)))
```

### 50. What are descriptors?

**Short answer:** A descriptor is an object defining `__get__`, `__set__`, or `__delete__` that controls attribute access on another class. `@property`, methods, and Django model fields are built on descriptors.

```python
class Positive:
    def __set_name__(self, owner, name):
        self.name = "_" + name
    def __get__(self, obj, objtype=None):
        if obj is None:                 # accessed on the class, e.g. Order.qty
            return self
        return getattr(obj, self.name)
    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError("Must be positive")
        setattr(obj, self.name, value)

class Order:
    qty = Positive()
    def __init__(self, qty):
        self.qty = qty

Order(2)      # OK
# Order(0)    -> ValueError
```

### 51. What is monkey patching?

**Short answer:** Changing a class, module, or function at runtime. Useful in tests (mocking) or hotfixing third-party code; dangerous in production because it's hidden, global, and breaks on library upgrades.

```python
import random
random.randint = lambda a, b: 4     # every caller now gets 4

# Safer in tests:
from unittest.mock import patch
with patch("random.randint", return_value=4):
    ...
```

### 52. What is the walrus operator (:=)?

**Short answer:** An assignment expression (Python 3.8+) that assigns and returns a value in the same expression.

```python
while (line := f.readline()):
    process(line)

if (n := len(items)) > 10:
    print(f"Too many items: {n}")

results = [y for x in data if (y := transform(x)) is not None]
```

### 53. What is structural pattern matching (match/case)?

**Short answer:** Python 3.10+ syntax that matches values against patterns including literals, sequences, dicts, and classes, with destructuring.

```python
def handle(event):
    match event:
        case {"type": "order", "id": oid}:
            return f"Order {oid}"
        case {"type": "refund", "amount": amt} if amt > 1000:
            return "Large refund: needs approval"
        case [x, y]:
            return f"Point {x},{y}"
        case _:
            return "Unknown"
```

### 54. @dataclass(frozen=True) vs NamedTuple

**Short answer:** Both give lightweight immutable records. `NamedTuple` is a real tuple (indexable, unpackable, slightly lighter). Frozen dataclass isn't a tuple, but supports defaults, `field()`, `__post_init__`, and is more extensible.

```python
from typing import NamedTuple
from dataclasses import dataclass

class PointNT(NamedTuple):
    x: int
    y: int

@dataclass(frozen=True)
class PointDC:
    x: int
    y: int

p = PointNT(1, 2); print(p[0], *p)     # tuple behaviour
q = PointDC(1, 2)
# q.x = 5 -> FrozenInstanceError
```

### 55. What are Enums, and why use them?

**Short answer:** An Enum is a set of named constant values. It prevents typos and invalid values, groups related constants, and makes code self-documenting.

```python
from enum import Enum

class OrderStatus(Enum):
    PENDING = "pending"
    SHIPPED = "shipped"
    DELIVERED = "delivered"

status = OrderStatus("shipped")
if status is OrderStatus.SHIPPED:
    print(status.name, status.value)     # SHIPPED shipped
```

## Part 6: Exceptions

### 56. Explain try, except, else, and finally.

**Short answer:** `try` runs risky code; `except` handles errors; `else` runs only if no exception happened; `finally` always runs (cleanup).

```python
try:
    user = fetch_user(uid)
except UserNotFound:
    user = None
else:
    log("Fetched successfully")     # only if no exception
finally:
    db.close()                      # always
```

### 57. How do you create a custom exception?

**Short answer:** Subclass `Exception` (or a more specific built-in). Add fields for context.

```python
class InsufficientBalance(Exception):
    def __init__(self, needed, available):
        self.needed, self.available = needed, available
        super().__init__(f"Need {needed}, have {available}")

def withdraw(balance, amt):
    if amt > balance:
        raise InsufficientBalance(amt, balance)
```

### 58. raise vs raise from

**Short answer:** `raise X from Y` chains exceptions, recording `Y` as the direct cause (`__cause__`) so tracebacks show both. `raise X from None` hides the original.

```python
try:
    data = json.loads(payload)
except json.JSONDecodeError as e:
    raise InvalidRequest("Bad JSON body") from e
```

### 59. Why is a bare except: bad practice?

**Short answer:** It catches everything, including `KeyboardInterrupt` and `SystemExit`, and hides bugs. Catch specific exceptions, or at most `Exception`, and log them.

```python
# Bad
try: process()
except: pass

# Good
try:
    process()
except (TimeoutError, ConnectionError) as e:
    logger.warning("Retrying: %s", e)
```

### 60. Exception vs Error in Python's hierarchy

**Short answer:** Python uses classes for both. `BaseException` is the root. `Exception` is the base for normal catchable errors; names ending in "Error" (`ValueError`, `TypeError`) are subclasses of `Exception`. `SystemExit`, `KeyboardInterrupt`, `GeneratorExit` inherit directly from `BaseException` so normal handlers don't catch them.

```text
BaseException
 ├── SystemExit
 ├── KeyboardInterrupt
 ├── GeneratorExit
 └── Exception
      ├── ValueError
      ├── TypeError
      ├── LookupError
      │    └── KeyError
      └── ... your custom exceptions
```


## Part 7: Concurrency and Async

### 61. Concurrency vs parallelism

**Short answer:** Concurrency is dealing with many tasks by switching between them (progress overlaps). Parallelism is executing many tasks at the same instant on multiple cores.

**Real-life example:** One chef juggling three dishes = concurrency. Three chefs cooking at once = parallelism.

### 62. What is an event loop?

**Short answer:** The core of asyncio: a loop that runs tasks, and when one awaits I/O, switches to another ready task, resuming the first when its I/O completes.

```python
import asyncio

async def task(name, delay):
    await asyncio.sleep(delay)   # yields control to the loop
    print(name, "done")

async def main():
    await asyncio.gather(task("A", 2), task("B", 1))

asyncio.run(main())
# B done, A done (total ~2s, not 3s)
```

### 63. async def, await, and asyncio.gather()

**Short answer:** `async def` defines a coroutine. `await` pauses it until an awaitable finishes. `asyncio.gather()` runs several coroutines concurrently and returns their results in order.

```python
import asyncio, httpx

async def get_status(client, url):
    r = await client.get(url)
    return r.status_code

async def main(urls):
    async with httpx.AsyncClient() as client:
        return await asyncio.gather(*(get_status(client, u) for u in urls))
```

### 64. What happens if you call blocking code inside an async function?

**Short answer:** It blocks the whole event loop; every other request waits. Fix: use async libraries, or offload to a thread with `asyncio.to_thread` / `run_in_executor`. In FastAPI, a plain `def` endpoint runs in a threadpool automatically.

```python
import asyncio, time

async def bad():
    time.sleep(5)                      # freezes everything

async def good():
    await asyncio.to_thread(time.sleep, 5)   # runs in a thread
```

### 65. What is a race condition? How do you prevent it?

**Short answer:** A bug where the result depends on the timing of threads accessing shared data. Prevent with locks, atomic operations, or DB-level locking (`select_for_update`).

```python
import threading

balance = 0
lock = threading.Lock()

def deposit():
    global balance
    for _ in range(100_000):
        with lock:                # only one thread at a time
            balance += 1
```

### 66. What is a deadlock?

**Short answer:** Two or more threads wait forever for locks held by each other. Avoid by acquiring locks in a consistent order, using timeouts, or minimising lock scope.

**Real-life example:** Two cars on a one-lane bridge from opposite ends, each waiting for the other to reverse.

```python
# Thread 1: lock_a then lock_b
# Thread 2: lock_b then lock_a   -> possible deadlock
# Fix: both acquire lock_a first, then lock_b
```

### 67. How does Celery work? What is a broker?

**Short answer:** Celery runs tasks in the background on worker processes. The app pushes a task message to a broker (Redis/RabbitMQ); workers pull and execute it; results can be stored in a result backend.

**Real-life example:** A restaurant: the waiter (web app) puts the order slip on the rail (broker); cooks (workers) pick it up, so the waiter can serve others.

```python
from celery import Celery
app = Celery("shop", broker="redis://localhost:6379/0")

@app.task(bind=True, max_retries=3)
def send_invoice(self, order_id):
    try:
        email_invoice(order_id)
    except SMTPError as e:
        raise self.retry(exc=e, countdown=60)

send_invoice.delay(42)       # returns immediately
```

**Interview tip:** Make tasks idempotent; they may run more than once.

## Part 8: Modules, Packaging, Environment

### 68. Module vs package

**Short answer:** A module is a single `.py` file. A package is a directory of modules (traditionally with `__init__.py`) that can be imported as a unit.

```text
shop/
  __init__.py
  orders.py          # module
  payments/          # sub-package
    __init__.py
    razorpay.py
```

```python
from shop.payments.razorpay import charge
```

### 69. What is __init__.py for?

**Short answer:** Marks a directory as a regular package, runs on first import, and can expose a cleaner public API or set `__all__`. Since Python 3.3 it's optional (namespace packages), but still recommended.

```python
# shop/payments/__init__.py
from .razorpay import charge
__all__ = ["charge"]

# Now callers can do:
from shop.payments import charge
```

### 70. What does if __name__ == "__main__": do?

**Short answer:** Code under it runs only when the file is executed directly, not when it's imported. Also required for `multiprocessing` when the `spawn` start method is used (the default on Windows and macOS).

```python
def main():
    print("Running report")

if __name__ == "__main__":
    main()     # runs for: python report.py, not for: import report
```

### 71. What is a virtual environment, and why use one?

**Short answer:** An isolated Python environment with its own packages, so projects don't conflict over versions.

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install django==5.0
pip freeze > requirements.txt
```

### 72. How do you manage dependencies?

**Short answer:** `requirements.txt` with pinned versions (pip), or modern tools like Poetry / uv with `pyproject.toml` and a lock file for reproducible builds. Separate dev dependencies (pytest, ruff) from production.

```toml
# pyproject.toml (Poetry)
[tool.poetry.dependencies]
python = "^3.11"
django = "^5.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"
```

### 73. What are circular imports, and how do you fix them?

**Short answer:** Module A imports B while B imports A, so one sees a partially initialised module (`ImportError` / `AttributeError`). Fix by moving shared code to a third module, importing inside the function, importing the module instead of names, or using `TYPE_CHECKING` for type-only imports.

```python
# orders.py
def create_order():
    from payments import charge      # local import breaks the cycle
    charge()

from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from payments import Payment     # only for type hints
```

## Part 9: Testing and Code Quality

### 74. How do you write unit tests with pytest?

**Short answer:** Write functions starting with `test_` and use plain `assert`. pytest discovers and runs them, with detailed failure output.

```python
# calc.py
def apply_gst(price):
    return round(price * 1.18, 2)

# test_calc.py
import pytest
from calc import apply_gst

def test_apply_gst():
    assert apply_gst(100) == 118.0

@pytest.mark.parametrize("price,expected", [(0, 0), (50, 59.0)])
def test_many(price, expected):
    assert apply_gst(price) == expected

def test_error():
    with pytest.raises(TypeError):
        apply_gst("abc")
```

### 75. What are fixtures in pytest?

**Short answer:** Reusable setup/teardown functions injected into tests by name. Scope can be function, module, or session.

```python
import pytest

@pytest.fixture
def user():
    u = User.objects.create(name="Asha")
    yield u                  # test runs here
    u.delete()               # teardown

def test_user_name(user):
    assert user.name == "Asha"
```

### 76. What is mocking? How do you use unittest.mock.patch?

**Short answer:** Replacing a real dependency (API, DB, time) with a fake object to test code in isolation. Patch where the name is used, not where it's defined.

```python
from unittest.mock import patch

# orders.py: from payments import charge
def place_order(amount):
    return "ok" if charge(amount) else "failed"

@patch("orders.charge", return_value=True)
def test_place_order(mock_charge):
    assert place_order(500) == "ok"
    mock_charge.assert_called_once_with(500)
```

### 77. Unit vs integration vs end-to-end tests

**Short answer:** Unit tests check one function/class in isolation (fast, many). Integration tests check components working together (API + DB). End-to-end tests check the full user flow through the UI (slow, few). This forms the test pyramid.

**Real-life example:** Testing a single brake (unit), the brake with the wheel (integration), then a test drive (E2E).

### 78. What is PEP 8? Which linters and formatters do you use?

**Short answer:** PEP 8 is Python's official style guide (naming, indentation, line length, imports). Common tools: black (formatter), ruff (fast linter + formatter), flake8, isort (imports), mypy (type checking), usually run via pre-commit hooks and CI.

```python
# Naming: snake_case functions/vars, PascalCase classes, UPPER_CASE constants
MAX_RETRIES = 3
class OrderService:
    def get_order(self, order_id): ...
```

```bash
ruff check . && black . && mypy .
```

## Part 10: Performance

### 79. How do you profile slow Python code?

**Short answer:** Measure first. `timeit` for small snippets, `cProfile` for which functions take time, `line_profiler` for line-level, `tracemalloc`/`memory_profiler` for memory, and APM tools in production.

```bash
python -m cProfile -s cumtime app.py
```

```python
import timeit
print(timeit.timeit("sum(range(1000))", number=10_000))
```

```python
import cProfile, pstats
cProfile.run("generate_report()", "out.prof")
pstats.Stats("out.prof").sort_stats("cumtime").print_stats(10)
```

### 80. How would you optimise a slow loop?

**Short answer:** Use better data structures (set/dict lookups), move work out of the loop, use comprehensions and built-ins (`sum`, `max`), cache results, batch DB/API calls, vectorise with NumPy/Pandas, or parallelise.

```python
# Slow: O(n*m) with list lookup + query per item
for o in orders:
    if o.user_id in blocked_list:
        ...
# Fast: O(n)
blocked = set(blocked_list)
result = [o for o in orders if o.user_id not in blocked]
```

### 81. Why are list comprehensions usually faster than loops with append()?

**Short answer:** Comprehensions run in optimised bytecode and avoid repeatedly looking up and calling the `append` method on each iteration.

```python
# Loop
result = []
for x in range(1_000_000):
    result.append(x * 2)

# Comprehension: typically ~20-40% faster
result = [x * 2 for x in range(1_000_000)]
```

### 82. String concatenation in a loop vs "".join()

**Short answer:** Strings are immutable, so `s += piece` in a loop may create a new string each time (O(n²) worst case). `"".join(list)` computes the total size once and builds a single string (O(n)).

```python
# Slow
s = ""
for row in rows:
    s += row + "\n"

# Fast
s = "\n".join(rows)
```

### 83. When would you use NumPy or Pandas instead of plain Python?

**Short answer:** For large numeric or tabular data. NumPy does vectorised operations in C on contiguous arrays (10–100x faster). Pandas adds labelled DataFrames for filtering, grouping, joins, and time series. Use PySpark when data exceeds one machine's memory.

```python
import numpy as np, pandas as pd

prices = np.array([100, 200, 300])
print(prices * 1.18)                     # no Python loop

df = pd.read_csv("orders.csv")
print(df.groupby("city")["amount"].sum())
```


## Part 11: Coding Questions

### 84. Reverse a string without using [::-1]

**Short answer:** Build the result from the end, or use reversed(). Mention two-pointer swapping for in-place reversal of lists.

```python
def reverse_string(s):
    result = []
    for ch in s:
        result.insert(0, ch)      # or iterate backwards
    return "".join(result)

def reverse_string2(s):
    return "".join(reversed(s))

print(reverse_string("python"))     # nohtyp
```

### 85. Check if a string is a palindrome

**Short answer:** Normalise (lowercase, keep alphanumerics), then compare with two pointers. O(n) time, O(1) extra space.

```python
def is_palindrome(s):
    s = [c.lower() for c in s if c.isalnum()]
    left, right = 0, len(s) - 1
    while left < right:
        if s[left] != s[right]:
            return False
        left += 1
        right -= 1
    return True

print(is_palindrome("A man, a plan, a canal: Panama"))    # True
```

### 86. Check if two strings are anagrams

**Short answer:** Same characters with the same counts. Counter is O(n); sorting is O(n log n).

```python
from collections import Counter

def is_anagram(a, b):
    return Counter(a.replace(" ", "").lower()) == Counter(b.replace(" ", "").lower())

print(is_anagram("listen", "silent"))   # True
```

### 87. Find the first non-repeating character

**Short answer:** Count characters in one pass, then return the first with count 1. O(n).

```python
from collections import Counter

def first_unique(s):
    counts = Counter(s)
    for ch in s:
        if counts[ch] == 1:
            return ch
    return None

print(first_unique("swiss"))     # w
```

### 88. Count the frequency of each character or word

**Short answer:** Use collections.Counter, or a dict with get.

```python
from collections import Counter

text = "the cat and the hat"
print(Counter(text.split()).most_common(2))    # [('the', 2), ('cat', 1)]

freq = {}
for ch in "banana":
    freq[ch] = freq.get(ch, 0) + 1
print(freq)   # {'b': 1, 'a': 3, 'n': 2}
```

### 89. Find duplicates in a list

**Short answer:** Track seen items in a set; anything seen twice is a duplicate. O(n).

```python
def find_duplicates(items):
    seen, dupes = set(), set()
    for x in items:
        if x in seen:
            dupes.add(x)
        else:
            seen.add(x)
    return list(dupes)

print(find_duplicates([1, 2, 3, 2, 4, 1]))    # [1, 2]
```

### 90. Find the second-largest number in a list

**Short answer:** One pass tracking the largest and second-largest distinct values. O(n), handles duplicates.

```python
def second_largest(nums):
    first = second = float("-inf")
    for n in nums:
        if n > first:
            first, second = n, first
        elif first > n > second:
            second = n
    return second if second != float("-inf") else None

print(second_largest([10, 20, 20, 5]))     # 10
```

### 91. Two Sum: find two numbers that add up to a target

**Short answer:** Store each number's index in a dict; for each number check if target - n was seen. O(n).

```python
def two_sum(nums, target):
    seen = {}
    for i, n in enumerate(nums):
        need = target - n
        if need in seen:
            return [seen[need], i]
        seen[n] = i
    return []

print(two_sum([2, 7, 11, 15], 9))     # [0, 1]
```

### 92. Flatten a nested list like [1, [2, [3, 4]], 5]

**Short answer:** Recursion (or a generator with yield from); an explicit stack avoids recursion limits.

```python
def flatten(lst):
    result = []
    for x in lst:
        if isinstance(x, list):
            result.extend(flatten(x))
        else:
            result.append(x)
    return result

print(flatten([1, [2, [3, 4]], 5]))     # [1, 2, 3, 4, 5]
```

### 93. Merge two sorted lists

**Short answer:** Two pointers, always take the smaller head. O(n + m).

```python
def merge_sorted(a, b):
    i = j = 0
    result = []
    while i < len(a) and j < len(b):
        if a[i] <= b[j]:
            result.append(a[i]); i += 1
        else:
            result.append(b[j]); j += 1
    result.extend(a[i:])
    result.extend(b[j:])
    return result

print(merge_sorted([1, 3, 5], [2, 4, 6]))    # [1, 2, 3, 4, 5, 6]
```

### 94. Remove duplicates while preserving order

**Short answer:** Use a seen set, or dict.fromkeys (dicts keep insertion order).

```python
def dedupe(items):
    seen = set()
    return [x for x in items if not (x in seen or seen.add(x))]

print(list(dict.fromkeys([3, 1, 3, 2, 1])))     # [3, 1, 2]
```

### 95. Find the missing number in a list of 1 to n

**Short answer:** Expected sum n(n+1)/2 minus the actual sum. O(n) time, O(1) space. (XOR also works and avoids overflow in other languages.)

```python
def missing_number(nums, n):
    return n * (n + 1) // 2 - sum(nums)

print(missing_number([1, 2, 4, 5], 5))    # 3
```

### 96. Rotate a list by k positions

**Short answer:** Use slicing with k % len to handle k larger than the list. collections.deque.rotate is another option.

```python
def rotate_right(lst, k):
    if not lst:
        return lst
    k %= len(lst)
    return lst[-k:] + lst[:-k]

print(rotate_right([1, 2, 3, 4, 5], 2))    # [4, 5, 1, 2, 3]

from collections import deque
d = deque([1, 2, 3, 4, 5]); d.rotate(2)    # same result
```

### 97. Group words that are anagrams of each other

**Short answer:** Use the sorted word as a dict key. O(n · k log k).

```python
from collections import defaultdict

def group_anagrams(words):
    groups = defaultdict(list)
    for w in words:
        groups["".join(sorted(w))].append(w)
    return list(groups.values())

print(group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"]))
# [['eat', 'tea', 'ate'], ['tan', 'nat'], ['bat']]
```

### 98. Check for balanced parentheses ({[]})

**Short answer:** Push opening brackets onto a stack; on a closing bracket, the top must match. Stack must be empty at the end. O(n).

```python
def is_balanced(s):
    pairs = {")": "(", "]": "[", "}": "{"}
    stack = []
    for ch in s:
        if ch in "([{":
            stack.append(ch)
        elif ch in pairs:
            if not stack or stack.pop() != pairs[ch]:
                return False
    return not stack

print(is_balanced("({[]})"), is_balanced("([)]"))     # True False
```

### 99. Longest substring without repeating characters

**Short answer:** Sliding window with a dict of last-seen indexes. O(n).

```python
def longest_unique(s):
    last_seen = {}
    start = best = 0
    for i, ch in enumerate(s):
        if ch in last_seen and last_seen[ch] >= start:
            start = last_seen[ch] + 1
        last_seen[ch] = i
        best = max(best, i - start + 1)
    return best

print(longest_unique("abcabcbb"))     # 3 ("abc")
```

### 100. Implement an LRU cache (with and without OrderedDict)

**Short answer:** Keep the most recently used item at the end; evict from the front when over capacity. Both get and put are O(1).

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity):
        self.cap = capacity
        self.data = OrderedDict()

    def get(self, key):
        if key not in self.data:
            return -1
        self.data.move_to_end(key)
        return self.data[key]

    def put(self, key, value):
        if key in self.data:
            self.data.move_to_end(key)
        self.data[key] = value
        if len(self.data) > self.cap:
            self.data.popitem(last=False)     # evict least recent

# Without OrderedDict: plain dict keeps insertion order (3.7+)
class LRUCache2:
    def __init__(self, capacity):
        self.cap, self.data = capacity, {}
    def get(self, key):
        if key not in self.data:
            return -1
        val = self.data.pop(key)    # re-insert to mark as recent
        self.data[key] = val
        return val
    def put(self, key, value):
        self.data.pop(key, None)
        self.data[key] = value
        if len(self.data) > self.cap:
            del self.data[next(iter(self.data))]
```

**Interview tip:** In an interview they may want the classic version: a dict + doubly linked list.

### 101. Write a Fibonacci generator

**Short answer:** Keep the last two values and yield forever (or up to n).

```python
def fibonacci(limit=None):
    a, b = 0, 1
    count = 0
    while limit is None or count < limit:
        yield a
        a, b = b, a + b
        count += 1

print(list(fibonacci(8)))   # [0, 1, 1, 2, 3, 5, 8, 13]
```

### 102. Sort a list of dictionaries by a key

**Short answer:** Use sorted with key; operator.itemgetter is slightly faster and supports multiple keys.

```python
from operator import itemgetter

employees = [
    {"name": "Asha", "dept": "Eng", "salary": 90000},
    {"name": "Ravi", "dept": "Eng", "salary": 120000},
    {"name": "Meena", "dept": "HR", "salary": 70000},
]

by_salary = sorted(employees, key=lambda e: e["salary"], reverse=True)
by_dept_then_name = sorted(employees, key=itemgetter("dept", "name"))
```

### 103. Merge two dictionaries and sum values for common keys

**Short answer:** Loop with get, or use Counter addition (drops non-positive results).

```python
from collections import Counter

a = {"apple": 3, "banana": 2}
b = {"banana": 5, "cherry": 1}

merged = dict(a)
for k, v in b.items():
    merged[k] = merged.get(k, 0) + v
print(merged)                  # {'apple': 3, 'banana': 7, 'cherry': 1}

print(dict(Counter(a) + Counter(b)))   # same result
print(a | b)    # plain merge (3.9+), b's values win: banana = 5
```

### 104. Write a decorator that caches function results

**Short answer:** Store results in a dict keyed by the arguments.

```python
import functools

def memoize(func):
    cache = {}
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        key = (args, tuple(sorted(kwargs.items())))
        if key not in cache:
            cache[key] = func(*args, **kwargs)
        return cache[key]
    return wrapper

@memoize
def slow_square(n):
    import time; time.sleep(1)
    return n * n

slow_square(4)       # 1 second
slow_square(4)       # instant
```

### 105. Write a context manager that measures execution time

**Short answer:** Record start time on enter, print elapsed on exit (inside finally so it runs even on errors).

```python
import time
from contextlib import contextmanager

@contextmanager
def timer(label="block"):
    start = time.perf_counter()
    try:
        yield
    finally:
        print(f"{label} took {time.perf_counter() - start:.3f}s")

class Timer:
    def __enter__(self):
        self.start = time.perf_counter()
        return self
    def __exit__(self, *exc):
        self.elapsed = time.perf_counter() - self.start
        print(f"Took {self.elapsed:.3f}s")

with timer("export"):
    export_csv()
```

## Part 12: Output-Based Trick Questions

### 106. What does this print?

```python
a = [1, 2, 3]
b = a
b += [4]
print(a)
```

**Output:** `[1, 2, 3, 4]`

b = a makes both names refer to the same list, and += on a list mutates in place (it calls extend). Note: b = b + [4] would create a new list and leave a unchanged.

### 107. What does this print?

```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])
```

**Output:** `[2, 2, 2]`

Closures capture the variable i, not its value at creation (late binding). When called, i is already 2.

```python
# Fix: bind the current value as a default argument
funcs = [lambda i=i: i for i in range(3)]
print([f() for f in funcs])   # [0, 1, 2]
```

### 108. What does this print?

```python
t = (1, 2, [3, 4])
t[2].append(5)
print(t)
```

**Output:** `(1, 2, [3, 4, 5])`

The tuple is immutable (its references can't change), but the list it holds is mutable. Bonus trap: t[2] += [5] appends and raises TypeError, because += mutates the list and then tries to reassign the tuple slot.

### 109. What does this print?

```python
print(0.1 + 0.2 == 0.3)
```

**Output:** `False`

Floats are stored in binary, and 0.1 and 0.2 can't be represented exactly; the sum is 0.30000000000000004.

```python
import math
from decimal import Decimal

print(math.isclose(0.1 + 0.2, 0.3))              # True
print(Decimal("0.1") + Decimal("0.2") == Decimal("0.3"))   # True
```

**Interview tip:** Use Decimal for money, never float.

### 110. What does this print?

```python
x = 10
def f():
    print(x)
    x = 5
f()
```

**Output:** `UnboundLocalError`

Because x is assigned inside f, Python treats x as local for the whole function (decided at compile time), so print(x) reads the local before it's assigned. Fix with global x, or rename the local variable.


