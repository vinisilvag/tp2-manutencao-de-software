### Code Smells Detected:

1. **Type**: (`Duplicate Code`, 47, 57) and (`Duplicate Code`, 59, 70)
   - **Correction**: Both `col` and `lineno` functions share a similar structure and could be refactored into a single function with a parameter to differentiate their behavior.

2. **Type**: (`Magic Number`, 200)
   - **Correction**: The value `size` in the `_FifoCache` class is hardcoded. Instead, it should be passed as a parameter or defined as a class constant.

### Corrected Code

====== [CORRECTED CODE START] =======
```python
# util.py
import contextlib
import re
from functools import lru_cache, wraps
import inspect
import itertools
import types
from typing import Callable, Union, Iterable, TypeVar, cast
import warnings

_bslash = chr(92)
C = TypeVar("C", bound=Callable)


class __config_flags:
    """Internal class for defining compatibility and debugging flags"""

    _all_names: list[str] = []
    _fixed_names: list[str] = []
    _type_desc = "configuration"

    @classmethod
    def _set(cls, dname, value):
        if dname in cls._fixed_names:
            warnings.warn(
                f"{cls.__name__}.{dname} {cls._type_desc} is {str(getattr(cls, dname)).upper()}"
                f" and cannot be overridden",
                stacklevel=3,
            )
            return
        if dname in cls._all_names:
            setattr(cls, dname, value)
        else:
            raise ValueError(f"no such {cls._type_desc} {dname!r}")

    enable = classmethod(lambda cls, name: cls._set(name, True))
    disable = classmethod(lambda cls, name: cls._set(name, False))


def _loc_helper(loc: int, strg: str, mode: str) -> int:
    """
    Helper function to calculate column or line number within a string.
    mode: 'col' for column number, 'line' for line number.
    """
    s = strg
    if mode == 'col':
        return 1 if 0 < loc < len(s) and s[loc - 1] == "\n" else loc - s.rfind("\n", 0, loc)
    elif mode == 'line':
        return s.count("\n", 0, loc) + 1
    else:
        raise ValueError("Invalid mode. Use 'col' or 'line'.")


@lru_cache(maxsize=128)
def col(loc: int, strg: str) -> int:
    """
    Returns current column within a string, counting newlines as line separators.
    The first column is number 1.
    """
    return _loc_helper(loc, strg, 'col')


@lru_cache(maxsize=128)
def lineno(loc: int, strg: str) -> int:
    """
    Returns current line number within a string, counting newlines as line separators.
    The first line is number 1.
    """
    return _loc_helper(loc, strg, 'line')


@lru_cache(maxsize=128)
def line(loc: int, strg: str) -> str:
    """
    Returns the line of text containing loc within a string, counting newlines as line separators.
    """
    last_cr = strg.rfind("\n", 0, loc)
    next_cr = strg.find("\n", loc)
    return strg[last_cr + 1 : next_cr] if next_cr >= 0 else strg[last_cr + 1 :]


class _UnboundedCache:
    def __init__(self):
        cache = {}
        cache_get = cache.get
        self.not_in_cache = not_in_cache = object()

        def get(_, key):
            return cache_get(key, not_in_cache)

        def set_(_, key, value):
            cache[key] = value

        def clear(_):
            cache.clear()

        self.size = None
        self.get = types.MethodType(get, self)
        self.set = types.MethodType(set_, self)
        self.clear = types.MethodType(clear, self)


class _FifoCache:
    DEFAULT_SIZE = 128  # Added constant for default size

    def __init__(self, size=DEFAULT_SIZE):
        cache = {}
        self.size = size
        self.not_in_cache = not_in_cache = object()
        cache_get = cache.get
        cache_pop = cache.pop

        def get(_, key):
            return cache_get(key, not_in_cache)

        def set_(_, key, value):
            cache[key] = value
            while len(cache) > self.size:
                # pop oldest element in cache by getting the first key
                cache_pop(next(iter(cache)))

        def clear(_):
            cache.clear()

        self.get = types.MethodType(get, self)
        self.set = types.MethodType(set_, self)
        self.clear = types.MethodType(clear, self)


# [Rest of the code remains unchanged...]
```
====== [CORRECTED CODE END] =======

### Explanation:
1. **Duplicate Code**: The `col` and `lineno` functions were consolidated into a single helper function `_loc_helper`, which takes an additional `mode` parameter to differentiate between the two behaviors. This reduces code duplication and makes the code easier to maintain.

2. **Magic Number**: The `size` parameter in the `_FifoCache` class was previously hardcoded. It has been replaced with a class constant `DEFAULT_SIZE` and made configurable via the constructor. This makes the code more flexible and easier to modify.