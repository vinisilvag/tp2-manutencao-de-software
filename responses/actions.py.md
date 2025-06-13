Code Smells Detected:
None

The provided Python code snippet is well-structured and does not exhibit any significant code smells. The code is modular, follows good naming conventions, and avoids common pitfalls like magic numbers or duplicated code. It also includes appropriate type hints and comments for clarity.

However, if we were to suggest minor improvements for maintainability or readability, they would include:

====== [SUGGESTED IMPROVEMENTS START] =======
1. **Type Checking for `ParseAction`**:  
   Consider adding more robust type checking or validation for the `ParseAction` type to ensure it matches the expected call signatures. This could involve using `Protocol` or custom decorators.

====== [ADDED CODE] =======
from typing import Protocol

class ParseActionProtocol(Protocol):
    def __call__(self, s: str, l: int, t: ParseResults) -> Any: ...
    def __call__(self, l: int, t: ParseResults) -> Any: ...
    def __call__(self, t: ParseResults) -> Any: ...
    def __call__(self) -> Any: ...
====== [ADDED CODE] =======

2. **Documentation for `OnlyOnce`**:  
   The `OnlyOnce` class could benefit from a more detailed docstring explaining its use case and limitations.

====== [ADDED CODE] =======
class OnlyOnce:
    """
    Wrapper for parse actions, to ensure they are only called once.
    Note: parse action signature must include all 3 arguments: `s` (str), `l` (int), and `t` (ParseResults).

    Example usage:
        @OnlyOnce
        def parse_action(s: str, l: int, t: ParseResults) -> ParseResults:
            # Process tokens
            return t

    Raises:
        ParseException: If the action is called more than once without a reset.
    """
====== [ADDED CODE] =======

3. **Avoid Magic Strings**:  
   The error messages in `with_attribute` and `with_class` could be extracted into constants for consistency and easier maintenance.

====== [ADDED CODE] =======
ATTRIBUTE_NOT_FOUND_MSG = "no matching attribute {attrName}"
ATTRIBUTE_VALUE_MISMATCH_MSG = "attribute {attrName!r} has value {tokens_value!r}, must be {attrValue!r}" # noqa: E501
====== [ADDED CODE] =======

These changes are optional and would primarily enhance readability and maintainability rather than fix glaring issues.
====== [SUGGESTED IMPROVEMENTS END] =======