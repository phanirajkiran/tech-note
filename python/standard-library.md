Standard Library
================

# Built-in Functions

- `all(iterable)`: Return `True` if all elements of the iterable are true (or if the iterable is empty).
- `any(iterable)`: Return `True` if any element of the iterable is true. If the iterable is empty, return `False`.
- `bin(x)`, `hex(x)`, `oct(x)`

  ```python
  >>> bin(60)
  '0b111100'
  >>> hex(60)
  '0x3c'
  >>> oct(60)
  '074'
  ```
- `chr(i)`: Return a string of one character whose ASCII code is the integer i.

  ```python
  >>> chr(97)
  'a'
  >>> ord('a')
  97
  ```
- `@classmethod`: A class method receives the class as implicit first argument.

  ```python
  class C(object):
      @classmethod
      def f(cls, arg1, arg2, ...):
          ...
  ```
- `@staticmethod`: A static method does not receive an implicit first argument.

  ```python
  class C(object):
      @staticmethod
      def f(arg1, arg2, ...):
          ...
  ```
- `cmp(x, y)`: The return value is negative if x < y, zero if x == y and strictly positive if x > y.
- `divmod(a, b)`: For plain and long integers, the result is the same as `(a // b, a % b)`.
- `enumerate(sequence, start=0)`: Return an enumerate object.

  ```python
  >>> seq = ['A', 'B', 'C']
  >>> list(enumerate(seq))
  [(0, 'A'), (1, 'B'), (2, 'C')]
  >>> list(enumerate(seq, start=1))
  [(1, 'A'), (2, 'B'), (3, 'C')]
  ```
- `map(function, iterable)`: Apply function to every item of **iterable** and return a list of the results.

  ```python
  >>> map(lambda x: x**2, [1, 2, 3, 4, 5])
  [1, 4, 9, 16, 25]
  ```
- `reduce(function, iterable[, initializer])`: Apply function of two arguments cumulatively to the items of iterable, from left to right, so as to reduce the iterable to a **single value**.

  ```python
  >>> reduce(lambda x, y: x+y, [1, 2, 3, 4, 5])
  15   # ((((1+2)+3)+4)+5)
  ```
- `zip([iterable, ...])`: Return a list of tuples, where the i-th tuple contains the i-th element from each of the argument sequences or iterables.

  ```python
  >>> x = [1, 2, 3]
  >>> y = [4, 5, 6]
  >>> zip(x, y)
  [(1, 4), (2, 5), (3, 6)]
  ```
- `filter(function, iterable)`: Construct a list from those elements of iterable for which function returns true.

  ```python
  >>> filter(lambda x: x % 5 == 0, range(1, 50))
  [5, 10, 15, 20, 25, 30, 35, 40, 45]
  ```
- `getattr(object, name[, default])`: `getattr(x, 'parent')` is equivalent to `x.parent`
- `max(iterable)`, `min(iterable)`, `sum(iterable)`
- `reversed(seq)`: Return a reverse **iterator**. seq must be an object which has a `__reversed__()` method or supports the sequence protocol (the `__len__()` method and the `__getitem__()` method with integer arguments starting at 0).
- `sorted(iterable[, cmp[, key[, reverse]]])`
- `super(C, self).method(arg)`: Only works for new-style classes.
- `xrange(stop)`, `xrange(start, stop[, step])`: Return an **xrange object** instead of a list. This is an opaque sequence type which yields the same values as the corresponding list, without actually storing them all simultaneously.

# Data Types

## datetime — Basic date and time types

- Main types: `datetime.date`, `datetime.time`, `datetime.datetime`, `datetime.timedelta`
- Get current date/time
  - `datetime.datetime.today()`
  - `datetime.datetime.now()`
  - `datetime.datetime.utcnow()`
- `replace()` method can be used to create new date instances

  ```python
  d1 = datetime.date(2015, 2, 14)
  d2 = d1.replace(year=2016)   # d1.year is not writable
  ```
- Combining Dates and Times

  ```python
  t = datetime.time(1, 2, 3)
  d = datetime.date.today()
  dt = datetime.datetime.combine(d, t)
  ```
- Formatting and Parsing

  ```python
  format = "%a %b %d %H:%M:%S %Y"
  s = datetime.datetime.today().strftime(format)   # formatting
  d = datetime.datetime.strptime(s, format)   # parsing
  ```
- Time Zones
  - abstract class `tzinfo`, no actual implementation
  - third-party package: [pytz](http://pytz.sourceforge.net/)

# Numeric and Mathematical Modules

## random — Generate pseudo-random numbers

- Generating Random Numbers
  - `random.random()`: return 0 <= n < 1.0
  - `random.uniform(min, max)`: return min + (max - min) * random()
  - `random.randint(min, max)`: return integer n in [min, max]
  - `random.randrange(min, max, step)`
- Seeding
  - `random.seed(hashable_object)`
  - e.g. `random.seed(1)`, `random.seed(time.time())`
- Random Generator State
  - `random.getstate()`
  - `random.setstate()`
- Picking Random Items

  ```python
  seq = [1, 2, 3, 4, 5]
  picked = random.choice(seq)
  ```
- Permutations

  ```python
  seq = [1, 2, 3, 4, 5]
  random.shuffle(seq)
  ```
- Class-level => **multiple** generators

  ```python
  seed = time.time()
  r = random.Random(seed)
  val = r.random()
  ```

# Cryptographic Services

## hashlib — Secure hashes and message digests

- "Backed" by OpenSSL, support md5, sha1, sha256...
  - check `hashlib.algorithms_available` for full set
- Sample usage

  ```python
  h = hashlib.md5()   # or h = hashlib.new('md5')
  h.update('put string here')
  hashed = h.hexdigest()
  ```
- Calling `update()` more than once
  - The `update()` method of the hash calculators can be called repeatedly. Each time, the digest is updated based on the additional text fed in.
  - `h.update(a); h.update(b)` is equivalent to `h.update(a+b)`

## hmac — Keyed-Hashing for Message Authentication

- Default algorithm is **md5**
- Sample usage

  ```python
  h = hmac.new('secret-shared-key’)
  h.update('put string here')
  hashed = h.hexdigest()
  ```
- If content is small

  ```python
  h = hmac.new('secret-shared-key', 'put string here', hashlib.md5)
  hashed = h.hexdigest()
  ```
